# Proxmox VM Maintenance Runner (Generic)

A **generic, sequential, interactive** maintenance + health audit runner for Linux VMs (Proxmox or anywhere).

You run it from your computer. It reads a simple cut sheet (`targets.yml`), processes VMs **one at a time**, and prints a summary of **OK vs FAILED**. At the end it gives you an interactive menu to re-run failed/all/one.

---

## What it does (per VM)

### SSH setup (first run)
- Ensures you have an SSH key (`~/.ssh/id_ed25519`)
- Installs your public key into the VM (`authorized_keys`)
- Pre-seeds `known_hosts` so password bootstrap works cleanly

### Updates (safe)
- Debian/Ubuntu: `apt update` + `apt upgrade (safe)` (no dist-upgrade)
- RHEL/Fedora: normal package updates
- No OS release upgrades

### Health checks (generic)
- systemd failed units
- disk usage threshold (default 90%, configurable)
- recent kernel error messages (`dmesg` tail)
- network: default route, DNS resolution, ping
- docker (if installed): daemon running, no unhealthy containers
- NVIDIA GPU (best effort): detects NVIDIA, checks module/dev nodes, runs `nvidia-smi` if present
- SMART (if available): flags disks reporting `FAILED`
- ZFS (if available): flags pools that are not healthy
- Optional app URL probe (if you provide `url` per target)

### Result
- If everything passes:
  - `hurray you didnt break anything...yet`
- If something fails:
  - The VM is marked FAILED, and the final summary shows what failed.

---

## Requirements (your computer)

Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y ansible openssh-client sshpass python3
pip3 install --user pyyaml
Notes:

sshpass is required for the first connection to a host when using password auth to install your SSH key.

After key bootstrap, SSH becomes key-based.

Project files
graphql
Copy code
proxmox-vm-maint/
├── run                  # Interactive batch runner + end menu
├── maintain_generic.yml # Ansible playbook (updates + health checks)
├── targets.yml          # Your cut sheet (list of targets)
└── README.md
Configure your cut sheet
Edit targets.yml:

yaml
Copy code
targets:
  - name: Jellyfin
    host: 192.168.8.123
    port: 22
    user: root
    url: "http://192.168.8.123:8096/"
url is optional (used for a quick HTTP reachability check from your computer).

Run
From the project directory:

bash
Copy code
./run
It will:

Ask for disk threshold

Run all targets one-by-one

Print a final summary (OK vs FAILED)

Present a menu:

Re-run failed

Re-run all

Run one by name

Exit

Results are stored in .last_results.json for the “re-run failed” option.

Common issues
Host key changed (reinstalled VM / reused IP)
bash
Copy code
ssh-keygen -R 192.168.8.123
SMART inside VMs
SMART often is not available inside a VM unless the disk/controller is passed through. The playbook skips SMART cleanly if smartctl is missing and only fails on explicit FAILED results.

Safety
No dist-upgrades

No OS release upgrades

Designed to be generic and portable

yaml
Copy code

---

## Quick setup commands (copy/paste)

```bash
mkdir -p proxmox-vm-maint && cd proxmox-vm-maint

# Create files (paste contents from this message):
# - targets.yml
# - run
# - maintain_generic.yml
# - README.md

chmod +x run

sudo apt update
sudo apt install -y ansible openssh-client sshpass python3
pip3 install --user pyyaml

./run

