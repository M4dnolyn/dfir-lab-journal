# Troubleshooting Log

Journal of technical incidents encountered during lab deployment, covering cross-cutting issues not attributable to a single component. Tool-specific issues are documented in their respective component files.

## INC-001 — VM Disk Saturation During Installation

**Components involved**: Docker, Timesketch, Splunk, VM filesystem

**Symptom**: OpenSearch transitions to `unhealthy` state and refuses to start. Container logs show:

```
[WARN] this node is unhealthy: health check failed on
       [/usr/share/opensearch/data/nodes/0]
```

Meanwhile, `df -h /` confirms saturation:

```
/dev/sda2    25G    25G    0    100%    /
```

**Root cause**: Initial 25 GB virtual disk was insufficient for the combined footprint: Timesketch Docker images (~6 GB), Splunk installation (~4 GB), forensic tools, and workspace for case files. OpenSearch refuses writes once available space becomes critical, manifesting as a health check failure.

**Resolution**:

1. **Clean stack shutdown**:
   ```bash
   cd ~/timesketch && sudo docker compose stop
   ```

2. **Expand virtual disk from host (VM powered off)**: VirtualBox → File → Tools → Virtual Media Manager → Select disk → Resize to 80 GB → Apply

3. **Extend partition and filesystem from VM**:
   ```bash
   sudo apt install cloud-guest-utils -y
   sudo growpart /dev/sda 2
   sudo resize2fs /dev/sda2
   df -h /   # expected: ~79 GB available
   ```

4. **Restart stack**:
   ```bash
   cd ~/timesketch && sudo docker compose up -d
   sudo docker compose ps   # wait for opensearch to reach "healthy"
   ```

**Lesson**: Size the virtual disk for the total environment footprint — not just the OS. For this lab, 80 GB is the minimum viable. OpenSearch disk saturation produces no explicit error message about the cause — the health check fails silently, making diagnosis less immediate.

---

## INC-002 — Insufficient Memory: VM Unusable Under Load

**Components involved**: Timesketch (OpenSearch), Splunk, VM

**Symptom**: Any command in the VM hangs for minutes. `uptime` shows abnormally high system load:

```
load average: 52.48, 25.98, 10.60
```

`free -h` confirms memory saturation:

```
Mem:    4.5Gi    4.4Gi    99Mi
Swap:   0B       0B       0B
```

**Root cause**: VM initially configured with 4.5 GB RAM and no swap. Timesketch (OpenSearch allocates 2 GB alone) and Splunk (~1.5 GB) running simultaneously exhausted available memory. Without swap, the Linux kernel cannot reclaim inactive memory pages — system load explodes and the system becomes practically unusable.

**Resolution**:

1. **Increase VM RAM (VM powered off)**: VirtualBox → Settings → System → Memory → 8192 MB

2. **Add swapfile for memory spikes**:
   ```bash
   sudo fallocate -l 4G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```

3. **Do not run Timesketch and Splunk simultaneously**:
   ```bash
   # Before starting Splunk
   cd ~/timesketch && sudo docker compose stop

   # Before starting Timesketch
   sudo -u splunk /opt/splunk/bin/splunk stop
   cd ~/timesketch && sudo docker compose up -d
   ```

**Lesson**: 8 GB RAM and 4 GB swap constitute the minimum for this lab. Timesketch + Splunk cohabitation on 8 GB is technically possible but leaves little margin for analysis tools. Start only services required for the current task.

---

## INC-003 — `vm.max_map_count` Not Persistent After Reboot

**Components involved**: OpenSearch, Linux kernel

**Symptom**: OpenSearch remains `unhealthy` after VM reboot, despite working correctly before. Logs indicate:

```
max virtual memory areas vm.max_map_count [65530] is too low,
increase to at least [262144]
```

**Root cause**: Kernel parameter `vm.max_map_count` controls the maximum number of memory map areas a process may use. Linux default (65530) is insufficient for OpenSearch (requires ≥262144). The `deploy_timesketch.sh` script sets this at runtime, but the change does not survive a reboot — it must be explicitly persisted. This behavior is not documented in the official Timesketch deployment guide.

**Resolution**:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl -p /etc/sysctl.d/99-opensearch.conf
```

**Verification after reboot**:

```bash
sysctl vm.max_map_count   # expected: vm.max_map_count = 262144
```

**Lesson**: Any kernel parameter modified at runtime (`sysctl -w`) must be explicitly persisted in `/etc/sysctl.d/` to survive reboots. The Timesketch deployment script does not do this automatically — it is a manual step to integrate systematically after initial deployment.

---

## Quick Diagnostic Matrix

In case of abnormal behavior at lab startup, run this verification sequence in order:

```bash
# 1. VM disk space
df -h /

# 2. Available memory
free -h

# 3. OpenSearch kernel parameter
sysctl vm.max_map_count

# 4. Timesketch container status
cd ~/timesketch && sudo docker compose ps

# 5. OpenSearch logs if unhealthy
sudo docker compose logs opensearch --tail 30

# 6. Splunk status
sudo -u splunk /opt/splunk/bin/splunk status
```

The vast majority of post-installation incidents on this lab are covered by one of these six checks.