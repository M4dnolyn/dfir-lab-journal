
01 — VM Provisioning
Objective

Deploy and configure the Ubuntu 24.04 LTS virtual machine that serves as the primary analysis workstation for all forensic investigations in this lab.
Prerequisites

    VirtualBox 7.x installed on the host machine
    Ubuntu 24.04 LTS ISO — ubuntu-24.04.5.1-desktop-amd64.iso (5.8 GB) → https://releases.ubuntu.com/noble/
    Minimum 40 GB free on the host disk before starting. The VM disk is dynamically allocated and will grow over time as tools, Docker images, and extracted evidence accumulate.

1. VM Creation
Specifications
Parameter 	Value 	Rationale
RAM 	8 192 MB 	Timesketch (OpenSearch) alone allocates 2 GB. Running it alongside Splunk requires 8 GB minimum to avoid swap thrashing.
vCPUs 	4 	Memory analysis and timeline generation are multi-threaded workloads.
Disk 	80 GB — dynamic 	Docker images for Timesketch stack alone consume ~6 GB. Evidence extraction and timeline output add several GB on top. Starting at 25 GB guarantees a painful resize mid-install.
Network 	NAT 	Sufficient isolation for a forensic analysis lab. The VM does not need to be reachable from the local network.
Type 	Linux / Ubuntu 64-bit 	—
What not to leave at default

VirtualBox defaults are tuned for general-purpose VMs, not forensic workloads :

    RAM : default is often 1–2 GB → set to 8 192 MB
    CPU : default is 1 → set to 4
    Disk : opt for dynamic allocation, size 80 GB
    Network : keep NAT

2. Ubuntu Installation

Standard graphical installation from the ISO. No special configuration required during the setup wizard, except :

    Partitioning : let Ubuntu manage automatically (guided install, single partition)
    Username : avoid accents and spaces in the session name
    Updates : accept if offered, or run after first boot :

sudo apt update && sudo apt upgrade -y

3. Guest Additions

Guest Additions are required for two critical lab functions :

    Shared folder — transfer of case files from host to VM
    Dynamic screen resizing — quality-of-life for extended analysis sessions

Why not apt install virtualbox-guest-additions-iso

This command pulls virtualbox, virtualbox-dkms, and virtualbox-qt as dependencies — packages intended for a host machine running VirtualBox, not for a guest VM. The result is an unresolvable dependency conflict. The virtual CD method below is the correct approach for guest machines.
Installation via VirtualBox virtual CD

# Step 1 — Install build dependencies
sudo apt update
sudo apt install build-essential dkms linux-headers-$(uname -r) -y

# Step 2 — Insert the virtual CD from the VirtualBox menu :
# Devices → Insert Guest Additions CD Image...

# Step 3 — Mount and run
sudo mount /dev/cdrom /mnt
sudo /mnt/VBoxLinuxAdditions.run

# Step 4 — Reboot
sudo reboot

Verification : after reboot, resize the VM window. The Ubuntu desktop should adapt dynamically to the new size. If it does not, the Guest Additions are not active — re-run Step 3.
4. Case File Transfer

Several approaches exist for moving case files from the host into the VM :
Method 	Effort 	Notes
Direct download inside VM 	Low 	Simplest if download links are available
USB key mounted in VM 	Low 	Works without network
VirtualBox shared folder 	Medium 	Permanent access, no duplication needed
SCP over SSH 	Low 	Requires SSH to be configured first (see section 5)

Approach used here : VirtualBox shared folder. Chosen because it provides persistent, read-write access to the original files without duplicating them, and does not depend on network configuration.
4.1 Create the shared folder in VirtualBox

VirtualBox menu → Devices → Shared Folders → Shared Folder Settings

Click + and configure :
Field 	Value
Folder path 	Path to the directory on the host containing the case files
Folder name 	LabCase (no spaces)
Auto-mount 	✅
Make permanent 	✅
Read-only 	☐
4.2 Grant access to the current user

sudo usermod -aG vboxsf $USER
sudo reboot

    The vboxsf group membership only takes effect after a full VM reboot, not after a simple session logout.

4.3 Verify access

ls /media/sf_LabCase/

VirtualBox automatically prepends sf_ to the folder name at the mount point.
4.4 Stage case files into the working directory

Rather than working directly from the shared folder mount point (which depends on VirtualBox being active), case files are copied to a local working directory :

mkdir -p ~/lab_case
cp /media/sf_LabCase/Triage.7z ~/lab_case/
cp /media/sf_LabCase/memdump.mem ~/lab_case/
cp /media/sf_LabCase/pagefile.sys ~/lab_case/
cp /media/sf_LabCase/traffic.pcapng ~/lab_case/
cp /media/sf_LabCase/Splunk_logs_export.csv ~/lab_case/
ls -lh ~/lab_case/

5. SSH Access from Host (optional)

This step is a convenience choice, not a lab requirement.

Working from the host terminal rather than the VM graphical interface removes friction when running long command sequences — particularly copy-pasting multi-line commands during tool installation and evidence processing.
5.1 Install and enable SSH server inside the VM

sudo apt install openssh-server -y
sudo systemctl enable --now ssh
sudo systemctl status ssh   # expected : active (running)

5.2 Configure port forwarding in VirtualBox

VirtualBox → VM Settings → Network → Advanced → Port Forwarding
Name 	Protocol 	Host port 	Guest port
SSH 	TCP 	2222 	22
5.3 Connect from the host

ssh $USER@localhost -p 2222

6. Swap File

Strongly recommended on an 8 GB VM — not optional in practice.

Without swap, OpenSearch (the Timesketch search backend) will saturate available memory during indexing peaks and freeze the VM. A 4 GB swapfile provides a sufficient safety margin.

sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

Verification :

free -h   # Swap line should show 4.0 Gi

    If fallocate fails with "no space left on device", the host disk is full. Free space on the host before retrying — see troubleshooting-log.md.

7. Snapshots (recommended)

Good practice — not a hard requirement, but saves significant time if something breaks during tool installation.

Two snapshots are recommended :
Name 	When to take it 	Purpose
base-os-clean 	After OS install + Guest Additions + shared folder, before any tool installation 	Clean OS baseline to roll back to if a tool install corrupts the system
lab-ready 	After all tools are installed and verified operational 	Working lab baseline — roll back here rather than reinstalling everything

VirtualBox → Machine → Take Snapshot
8. Issues Encountered
Guest Additions dependency conflict

Symptom : apt install virtualbox-guest-additions-iso fails with dependency errors involving virtualbox-dkms, virtualbox-qt, virtualbox-modules.

Root cause : these packages target a VirtualBox host, not a guest VM. The apt package pulls them in as dependencies, creating an unresolvable conflict in a guest context.

Resolution : use the virtual CD method exclusively (section 3). Clean up the failed install first if needed :

sudo apt remove --purge virtualbox virtualbox-qt virtualbox-dkms \
  virtualbox-guest-additions-iso -y
sudo apt --fix-broken install -y
sudo apt autoremove -y

VM disk saturated at 25 GB

Symptom : VirtualBox error VERR_DISK_FULL — VM freezes completely mid-session and cannot resume until disk space is freed.

Root cause : initial disk undersized at 25 GB. Docker images for the Timesketch stack, forensic tooling, and evidence files exceed this limit before analysis even begins.

Resolution :

    Power off the VM
    VirtualBox → File → Tools → Virtual Media Manager
    Select the VM disk → resize to 80 GB → Apply
    Boot the VM and extend the filesystem :

sudo apt install cloud-guest-utils -y
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
df -h /   # should now show ~79 GB total

Host disk saturated — VM frozen

Symptom : same VERR_DISK_FULL error, but the VM disk size is fine.

Root cause : the dynamically allocated .vdi file grew on the host until the host filesystem itself ran out of space. The VM freezes at the hypervisor level, not at the guest OS level.

Resolution : free space on the host (installation ISOs, downloaded .deb packages, unused VM snapshots). Target a minimum of 30 GB free on the host to absorb ongoing VM disk growth during analysis.
Expected State After Provisioning

✅ Ubuntu 24.04 LTS — 8 GB RAM, 80 GB disk, 4 vCPUs
✅ Guest Additions active (dynamic window resizing confirmed)
✅ Shared folder mounted at /media/sf_LabCase/
✅ Case files staged in ~/lab_case/
✅ SSH reachable from host on localhost:2222      [optional]
✅ 4 GB swapfile active and persistent
✅ Snapshots : base-os-clean / lab-ready

