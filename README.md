Proxmox VM Maintenance Runner (Generic)

This project is a generic, sequential, interactive maintenance and health-check runner for Linux virtual machines.
It works great for Proxmox VMs, but it’s not tied to Proxmox — any reachable Linux VM will work.

You run everything from your own computer.
The tool reads a simple “cut sheet” (targets.yml), connects to each VM one at a time, performs safe updates and health checks, and then gives you a clear OK vs FAILED summary.

At the end of a run, you get an interactive menu that lets you:

re-run only failed machines

re-run everything

re-run a single VM by name

exit cleanly

How this tool works (high-level)

You define your VMs in targets.yml

You run ./run

Each VM is processed sequentially

Results are summarized

You choose what to do next from a menu

Nothing runs in parallel, nothing auto-reboots, and nothing does OS upgrades.

What happens on each VM
1. SSH setup (first run only)

On the first connection to a VM, the tool:

Ensures you have an SSH key on your computer (~/.ssh/id_ed25519)

Installs your public SSH key into the VM’s authorized_keys

Automatically adds the VM to your known_hosts file
(prevents SSH fingerprint prompts from breaking automation)

After this step, future runs use key-based SSH instead of passwords.

2. Safe system updates

Only safe updates are performed.

Debian / Ubuntu

apt update

apt upgrade (safe)

autoclean + autoremove

❌ No dist-upgrade

RHEL / Fedora

Normal package updates

❌ No OS release upgrades (ever)

3. Health checks (generic, non-destructive)

The following checks are run automatically:

System

Failed systemd units

Disk usage warnings (default ≥ 90%, configurable)

Recent kernel errors (dmesg tail)

Network

Default route exists

DNS resolution works

Ping test (gateway or fallback IP)

Docker (only if installed)

Docker daemon running

No containers in an unhealthy state

NVIDIA GPU (best-effort)

Detects NVIDIA hardware

Checks kernel module and /dev/nvidia*

Runs nvidia-smi if available

SMART disk health (if available)

Detects smartctl

Flags disks reporting FAILED

Skips cleanly if SMART isn’t available (common in VMs)

ZFS (if available)

Checks zpool status

Flags degraded or unhealthy pools

Optional application URL check

If a url is provided for a VM, a simple HTTP check is done from your computer

Useful to confirm services like Jellyfin, Nextcloud, etc. are reachable

4. Result and reporting

If everything passes:

hurray you didnt break anything...yet


If anything fails:

The VM is marked FAILED

The final summary shows exactly which VMs failed

You can immediately re-run failed hosts from the menu

Requirements (on your computer)
Debian / Ubuntu
sudo apt update
sudo apt install -y ansible openssh-client sshpass python3
pip3 install --user pyyaml

Notes

sshpass is required only for the first connection when using password-based SSH to install your SSH key

After key bootstrap, all SSH connections are key-based

Project structure
proxmox-vm-maint/
├── run                  # Interactive batch runner + end menu
├── maintain_generic.yml # Ansible playbook (updates + health checks)
├── targets.yml          # Cut sheet (list of VMs)
└── README.md

Configure your cut sheet (targets.yml)

Edit targets.yml to describe your VMs:

targets:
  - name: Jellyfin
    host: 192.168.8.123
    port: 22
    user: root
    url: "http://192.168.8.123:8096/"

Fields explained

name – Friendly name shown in output and menus

host – IP or DNS name

port – SSH port (usually 22)

user – SSH user

url – (optional) HTTP endpoint to test

Running the tool

From the project directory:

./run


The runner will:

Ask for a disk-usage warning threshold

Run all targets one by one

Print a summary of OK vs FAILED

Present an interactive menu:

Re-run failed

Re-run all

Run one by name

Exit

Results from the last run are stored in:

.last_results.json


This enables the “re-run failed” option.

Common issues and fixes
SSH host key changed (VM reinstalled or IP reused)
ssh-keygen -R 192.168.8.123

SMART inside virtual machines

SMART data is often unavailable inside VMs unless disks or controllers are passed through.
The playbook:

Skips SMART checks if smartctl is missing

Only fails when SMART explicitly reports FAILED

Safety guarantees

❌ No dist-upgrades

❌ No OS release upgrades

❌ No automatic reboots

✔ Designed to be generic, portable, and conservative

Quick setup (copy & paste)
mkdir -p proxmox-vm-maint && cd proxmox-vm-maint

# Create these files:
# - targets.yml
# - run
# - maintain_generic.yml
# - README.md

chmod +x run

sudo apt update
sudo apt install -y ansible openssh-client sshpass python3
pip3 install --user pyyaml

./run

