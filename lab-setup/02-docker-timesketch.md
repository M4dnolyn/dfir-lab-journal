# 02 — Timesketch Deployment via Docker

## Objective

Deploy Timesketch, the collaborative forensic timeline analysis and visualization platform, using Docker as the runtime environment.

Timesketch relies on a stack of six containerized services:

| Service | Role |
|---------|------|
| `timesketch-web` | Main web application |
| `timesketch-worker` | Async import processing |
| `opensearch` | Event indexing and search engine |
| `postgres` | Relational database (users, sketches) |
| `redis` | Message queue between web and worker |
| `nginx` | HTTP/HTTPS reverse proxy |

## Prerequisites

- Operational Ubuntu 24.04 LTS VM (see [01-vm-provisioning.md](01-vm-provisioning.md))
- VM RAM: 8 GB minimum — OpenSearch alone allocates 2 GB at startup
- VM Disk: 80 GB — Docker images for the stack alone weigh ~6 GB
- Internet access from the VM for image downloads

## 1. Why the Official Docker Repository and Not `docker.io`

Ubuntu provides the `docker.io` package in its standard repositories — a community version maintained by Canonical. This package has two limitations for this lab:

- It does not provide `docker-compose-plugin` (the `docker compose` command integrated into the Docker client), only the legacy standalone `docker-compose` binary — which is itself unavailable in Ubuntu 24.04.
- It lags behind upstream versions: Docker 29.x is not available via `docker.io`.

The official Docker repository (<https://download.docker.com>) provides `docker-ce`, `docker-ce-cli`, `containerd.io`, and `docker-compose-plugin` in their latest versions, properly packaged for Ubuntu.

> ⚠️ If `docker.io` or `docker-compose` have already been installed, remove them before proceeding — they will conflict with `docker-ce`.

## 2. Docker Installation

### 2.1 Remove Conflicting Packages

```bash
sudo apt remove docker.io docker-compose containerd runc -y
sudo apt autoremove -y
```

### 2.2 Install Prerequisites

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg -y
```

### 2.3 Add Official Docker GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.gpg
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg /tmp/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**Verification** — the file must exist and weigh approximately 2,760 bytes:

```bash
ls -lh /etc/apt/keyrings/docker.gpg
```

### 2.4 Add Docker Repository

The following command must be entered in a single line — line breaks in an Ubuntu terminal can truncate the command and produce an empty `docker.list` file without visible error:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

**Verification** — the file must not be empty:

```bash
cat /etc/apt/sources.list.d/docker.list
# expected: deb [arch=amd64 signed-by=...] https://download.docker.com/linux/ubuntu noble stable
```

### 2.5 Install Docker

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin -y
```

### 2.6 Start and Enable at Boot

```bash
sudo systemctl enable --now docker
```

### 2.7 Verify

```bash
sudo docker run --rm hello-world
sudo docker compose version
```

The output `Hello from Docker!` confirms the daemon works and can pull images. `docker compose version` must show `v5.x.x`.

## 3. Timesketch Deployment

### 3.1 Download Official Deployment Script

```bash
cd ~
curl -s -O https://raw.githubusercontent.com/google/timesketch/master/contrib/deploy_timesketch.sh
chmod 755 deploy_timesketch.sh
```

> ⚠️ The URL must be copied exactly — any browser or terminal autocorrection (notably `githubusercontent` → `githubuserconsent`) makes the script inaccessible without an explicit error message.

### 3.2 Run the Script

```bash
sudo ./deploy_timesketch.sh
```

The script performs the following:

- Verifies Docker and docker compose presence
- Sets `vm.max_map_count` to 262144 (required by OpenSearch)
- Generates configuration files in `~/timesketch/`
- Downloads Docker images for the stack

At the prompt `Would you like to start the containers? [y/N]` — answer **N**. Containers will be started manually in the next step for better control of the startup sequence.

### 3.3 Permanent `vm.max_map_count` Setting

The script sets `vm.max_map_count` for the current session only. To persist after VM reboot:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl -p /etc/sysctl.d/99-opensearch.conf
```

**Verification**:

```bash
sysctl vm.max_map_count   # expected: vm.max_map_count = 262144
```

### 3.4 Start the Stack

```bash
cd ~/timesketch
sudo docker compose up -d
```

First startup downloads missing images (~1.5 GB). Duration varies by connection — allow 5 to 15 minutes.

Track progress:

```bash
sudo docker compose ps
```

Wait for all services to show `healthy` or `running` before continuing. OpenSearch is the slowest to start (30–60 seconds after others).

**Expected Result**:

```
NAME                STATUS
nginx               Up
opensearch          Up (healthy)
postgres            Up (healthy)
redis               Up (healthy)
timesketch-web      Up
timesketch-worker   Up
```

### 3.5 Create a User

Wait an additional minute after all services are Up to let `timesketch-web` finish initialization, then:

```bash
sudo docker compose exec timesketch-web tsctl create-user <username>
```

Enter and confirm a password. Expected message: `User account for <username> created/updated`

### 3.6 Web Interface Access

Open Firefox in the VM:

```
http://localhost
```

> ⚠️ Use `http://` not `https://` — no SSL certificate is configured by default. If the browser auto-redirects to HTTPS, open a private window (Ctrl+Shift+P) and re-enter `http://localhost`.

## 4. Daily Management Commands

### Start the Stack

```bash
cd ~/timesketch
sudo docker compose up -d
```

### Stop the Stack (End of Session)

```bash
cd ~/timesketch
sudo docker compose stop
```

### Check Service Status

```bash
sudo docker compose ps
```

### View Service Logs

```bash
sudo docker compose logs opensearch --tail 50
sudo docker compose logs timesketch-web --tail 50
```

### Restart a Specific Service

```bash
sudo docker compose restart opensearch
```

> ⚠️ Timesketch does not start automatically with the VM. Run `docker compose up -d` manually at the beginning of each session if Timesketch is required for the current analysis.

## 5. Resource Considerations

| Service | RAM Allocated | Note |
|---------|---------------|------|
| OpenSearch | 2 GB (configured by script) | Minimum functional value |
| timesketch-web + worker | ~500 MB | Variable under load |
| postgres + redis + nginx | ~300 MB | Stable, low variance |
| **Total stack** | **~2.8 GB** | On 8 GB VM RAM |

**Recommendation**: do not run Timesketch and Splunk simultaneously unless necessary. Splunk alone consumes ~1.5 GB additional — the remaining margin for OS and analysis tools becomes insufficient.

```bash
# Stop Timesketch before starting Splunk
cd ~/timesketch && sudo docker compose stop
sudo -u splunk /opt/splunk/bin/splunk start
```

## 6. Issues Encountered and Resolutions

### `docker-compose-plugin` Not Found via apt

**Symptom**:

```
E: Unable to locate package docker-compose-plugin
```

**Cause**: attempted installation from Ubuntu standard repositories, which do not provide this package.

**Resolution**: add the official Docker repository (<https://download.docker.com>) before any installation — see §2 of this document.

### `docker.service` Fails to Start via systemd

**Symptom**:

```
Job for docker.service failed because the control process exited with error code.
```

**Cause**: systemd starts too early after installation, conflicting with a legacy socket or residual `docker.io` instance. The `dockerd` daemon itself works correctly — the issue is isolated to systemd orchestration.

**Diagnosis**: run `sudo dockerd` directly to observe if the daemon starts without error. If it does, the problem is at the systemd level.

**Resolution**:

```bash
sudo systemctl reset-failed docker.service docker.socket
sudo systemctl restart containerd
sudo systemctl start docker
sudo systemctl status docker
```

### OpenSearch Remains Unhealthy Indefinitely

**Symptom**:

```
[WARN] this node is unhealthy: health check failed on
       [/usr/share/opensearch/data/nodes/0]
```

**Possible Causes** (by probability):

1. **VM disk saturated** — OpenSearch refuses to write if available space is insufficient.
   ```bash
   df -h /   # verify Use% < 90%
   ```

2. **`vm.max_map_count` too low** — OpenSearch requires 262144 minimum.
   ```bash
   sysctl vm.max_map_count
   sudo sysctl -w vm.max_map_count=262144   # if value < 262144
   ```

3. **Insufficient memory** — VM was initially at 4.5 GB, causing system load of 52 (normal load: 1–4). Resolution: increase VM RAM to 8 GB via VirtualBox.

**Recommended Diagnostic Sequence**:

```bash
df -h /
sysctl vm.max_map_count
free -h
sudo docker compose logs opensearch --tail 30
```

### `docker compose exec` Hangs (No Response)

**Symptom**: `sudo docker compose exec timesketch-web tsctl create-user` does not respond, Ctrl+C required to interrupt.

**Cause**: `timesketch-web` has not finished internal initialization, or VM lacks resources (RAM or CPU saturated).

**Resolution**: wait 2–3 additional minutes after all services are Up, then retry the command. Verify `free -h` to ensure available memory is sufficient (> 1 GB).

## ✅ Expected Final Deployment State

- [x] Docker 29.x installed from official repository
- [x] `docker compose version` v5.x.x
- [x] Timesketch stack: 6 services at Up/healthy state
- [x] Timesketch user created
- [x] Interface accessible at `http://localhost`
- [x] `vm.max_map_count = 262144` (persistent after reboot)
- [x] 4 GB swap active (see [01-vm-provisioning.md](01-vm-provisioning.md))