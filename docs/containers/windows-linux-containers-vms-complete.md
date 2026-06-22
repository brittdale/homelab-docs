# Windows & Linux Containers and VMs — Complete Mastery Reference

**Domain:** Container Orchestration, Virtualization, and Hybrid OS Management
**Scope:** Standalone operation, Kubernetes integration, Windows Registry, PowerShell command authority, and cross-platform connectivity
**Last Updated:** 2026-06-22
**File Path:** `windows-linux-containers-vms-complete.md`

---

## Table of Contents

1. [The Fundamental Distinction — VMs vs Containers](#1-the-fundamental-distinction--vms-vs-containers)
2. [Linux Containers (LXC/Docker) — How They Work](#2-linux-containers-lxcdocker--how-they-work)
3. [Linux VMs — How They Work](#3-linux-vms--how-they-work)
4. [Windows Containers — How They Work](#4-windows-containers--how-they-work)
5. [Windows VMs — How They Work](#5-windows-vms--how-they-work)
6. [WSL2 — The Hybrid Bridge](#6-wsl2--the-hybrid-bridge)
7. [Switching Between Linux and Windows Container Modes](#7-switching-between-linux-and-windows-container-modes)
8. [The Windows Registry — What It Is and Why It Matters for Containers](#8-the-windows-registry--what-it-is-and-why-it-matters-for-containers)
9. [PowerShell Command Authority — Complete Reference](#9-powershell-command-authority--complete-reference)
10. [Standalone Usage — Running Without Kubernetes](#10-standalone-usage--running-without-kubernetes)
11. [Kubernetes Integration — Linux Containers](#11-kubernetes-integration--linux-containers)
12. [Kubernetes Integration — Windows Containers](#12-kubernetes-integration--windows-containers)
13. [Kubernetes Integration — VMs (KubeVirt)](#13-kubernetes-integration--vms-kubevirt)
14. [Cross-Platform Connectivity — How Linux and Windows Nodes Communicate](#14-cross-platform-connectivity--how-linux-and-windows-nodes-communicate)
15. [Scheduling and Affinity — Keeping Workloads on the Right OS](#15-scheduling-and-affinity--keeping-workloads-on-the-right-os)
16. [Storage Boundaries Across OS Types](#16-storage-boundaries-across-os-types)
17. [Networking Differences Between Linux and Windows Containers](#17-networking-differences-between-linux-and-windows-containers)
18. [Troubleshooting Cross-Platform Failures](#18-troubleshooting-cross-platform-failures)
19. [Architectural Gotchas and Hard Rules](#19-architectural-gotchas-and-hard-rules)
20. [Homelab Practice Tasks](#20-homelab-practice-tasks)
- [Quick Reference Tables](#quick-reference-tables)

---

## 1. The Fundamental Distinction — VMs vs Containers

Before anything else, the distinction between a **Virtual Machine** and a **Container** must be airtight. They solve different problems and operate at different layers of the stack.

### Virtual Machines

A VM is a full emulation of a complete physical computer. It includes:

- A **virtual CPU** (backed by the hypervisor)
- **Virtual RAM** (carved from host memory)
- A **virtual disk** (a file like `.vhdx`, `.vmdk`, or `.qcow2` on the host)
- A **complete operating system** — its own kernel, init system, drivers, and userspace

The hypervisor (Hyper-V, KVM, VMware, Proxmox) sits between the hardware and the VMs. It presents fake hardware to each VM. The VM has zero knowledge it is not running on bare metal.

> 🎯 **Key rule:** Every VM has its own kernel. Two VMs on the same host run two entirely separate kernels. They are completely isolated from each other and from the host.

### Containers

A container is **not** a VM. It is an isolated process (or group of processes) running directly on the **host's kernel**. It does not have its own kernel. It does not emulate hardware.

What makes it feel isolated:

- **Linux namespaces** — each container gets its own view of the network, filesystem, PID list, user list, and hostname
- **cgroups (control groups)** — the kernel enforces CPU, memory, and I/O limits per container
- **Union filesystems** — container images are layered read-only filesystems with a writable layer on top

> ⚠️ **Critical consequence:** Because a container shares the host kernel, a Linux container **must** run on a Linux kernel. A Windows container **must** run on a Windows kernel. This is not a configuration choice — it is a hard physical boundary enforced by the CPU instruction set and system call interface.

### Side-by-Side Comparison

| Property | Virtual Machine | Container |
|----------|----------------|-----------|
| Has its own kernel | ✅ Yes | ❌ No — shares host kernel |
| Boot time | 10s – 3min | Milliseconds |
| Disk footprint | GBs (full OS) | MBs (app layers only) |
| Isolation level | Hardware-level | Process-level |
| Can run different OS than host | ✅ Yes (with hypervisor) | ❌ No (must match kernel family) |
| Resource overhead | High | Very low |
| Stateful by default | Yes | No (ephemeral by default) |
| Proxmox term | KVM (QEMU VM) | LXC (Linux Container) |
| Docker term | Not applicable | Container |
| Kubernetes term | Pod (via KubeVirt) | Pod |

---

## 2. Linux Containers (LXC/Docker) — How They Work

### The Two Linux Container Ecosystems

There are two distinct container ecosystems on Linux:

**LXC (Linux Containers)** — used by Proxmox. These are system containers. They run a nearly complete Linux userspace (init system, systemd, cron, SSH daemon) but share the Proxmox host kernel. They feel like lightweight VMs but are technically containers.

**Docker/OCI Containers** — used by Docker, Podman, Kubernetes. These are application containers. They run a single process (or a small process tree) in isolation. No init system, no SSH daemon — just your app.

### How a Linux Container Actually Works

When Docker runs `nginx:latest`:

```
[ nginx process ]
      │  makes system calls (read, write, socket, fork)
      ▼
[ Host Linux Kernel ]  ← the actual kernel handling everything
      │
[ Hardware ]
```

The container image contains:
- A minimal Linux userspace filesystem (from a base like `alpine`, `debian`, `ubuntu`)
- The application binary and its dependencies
- **No kernel** — the kernel is always the host's

### Linux Container Image Layers

```
┌──────────────────────────────────────────┐  ← Writable container layer (your runtime changes)
├──────────────────────────────────────────┤  ← App layer (nginx binary, config)
├──────────────────────────────────────────┤  ← Dependency layer (libssl, libc)
└──────────────────────────────────────────┘  ← Base OS layer (debian:slim filesystem)
                                              (read-only, shared across containers)
```

Each layer is a diff. The union filesystem (OverlayFS on modern Linux) stacks them. Multiple containers sharing the same base image share those read-only layers in memory and on disk.

### Proxmox LXC vs Docker — Key Differences

| Property | Proxmox LXC | Docker Container |
|----------|-------------|-----------------|
| Init system | systemd (full) | None (single process) |
| SSH access | Yes (typical) | No (exec in via `docker exec`) |
| Networking | Proxmox bridge (vmbr0) | Docker bridge (docker0) or host |
| Storage | Proxmox storage pool (ZFS, LVM, dir) | Docker volumes or bind mounts |
| Privileged mode | Optional (gives host kernel access) | Optional (`--privileged`) |
| Kubernetes use | Not directly (k3s agent runs inside) | Yes — Kubernetes manages Docker/containerd |
| Typical use | Full service instances (Gitea, Mattermost) | Microservice workloads |

### Running Linux Containers Standalone (Docker)

```bash
# Pull and run a container interactively
docker run -it ubuntu:22.04 bash

# Run detached with port mapping and volume
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -v /opt/stacks/gitea/data:/data \
  gitea/gitea:latest

# List running containers
docker ps

# Execute a command inside a running container
docker exec -it gitea bash

# View container logs
docker logs -f gitea

# Stop and remove
docker stop gitea && docker rm gitea

# Inspect container details (networking, mounts, env vars)
docker inspect gitea
```

### Docker Compose — Multi-Container Standalone

```yaml
# /opt/stacks/gitea/docker-compose.yml
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    ports:
      - "3000:3000"
      - "222:22"
    volumes:
      - ./data:/data
    environment:
      - USER_UID=1000
      - USER_GID=1000
```

```bash
# Start stack
docker compose up -d

# Tear down (keep volumes)
docker compose down

# Tear down and destroy volumes
docker compose down -v
```

---

## 3. Linux VMs — How They Work

### The Hypervisor Types

**Type 1 (Bare Metal)** — Proxmox KVM/QEMU. The hypervisor IS the operating system. It runs directly on hardware. VMs run on top of it. This is the typical homelab Proxmox setup.

**Type 2 (Hosted)** — VMware Workstation, VirtualBox, Hyper-V on Windows desktop. The hypervisor runs as an application inside a host OS.

### How KVM Works on Proxmox

```
[ VM: Ubuntu Worker Node ]   [ VM: PBS ]   [ VM: Gitea ]
         │                        │               │
         ▼                        ▼               ▼
[ QEMU — Hardware Emulation Layer ]
         │
         ▼
[ KVM — Kernel Virtual Machine (Linux kernel module) ]
         │
         ▼
[ Proxmox Host Kernel (Debian Linux) ]
         │
         ▼
[ Physical Hardware: CPU, RAM, NIC, Disk ]
```

KVM uses CPU virtualization extensions (Intel VT-x / AMD-V) to run guest code at near-native speed. QEMU handles device emulation (virtual NIC, virtual disk controller, virtual display).

### Linux VM Disk Formats

| Format | Extension | Used By | Notes |
|--------|-----------|---------|-------|
| QCOW2 | `.qcow2` | Proxmox/KVM | Supports snapshots, thin provisioning |
| VMDK | `.vmdk` | VMware | Can be imported into Proxmox |
| VHD/VHDX | `.vhdx` | Hyper-V | Microsoft's format |
| RAW | `.img` | Any | No overhead, no snapshot support |

### Creating a Linux VM on Proxmox (CLI)

```bash
# Create a VM with 4 cores, 8GB RAM, 50GB disk
qm create 200 \
  --name "k3s-worker-1" \
  --cores 4 \
  --memory 8192 \
  --net0 virtio,bridge=vmbr0 \
  --virtio0 local-lvm:50 \
  --ide2 local:iso/ubuntu-22.04-live-server-amd64.iso,media=cdrom \
  --boot order=ide2;virtio0 \
  --ostype l26

# Start the VM
qm start 200

# Check status
qm status 200

# Stop cleanly
qm shutdown 200

# Hard stop
qm stop 200

# Snapshot
qm snapshot 200 clean-install "Before k3s agent"

# Rollback
qm rollback 200 clean-install
```

### Proxmox LXC Container (CLI)

```bash
# Create an LXC container
pct create 300 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname mattermost \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --start 1

# Enter the container
pct enter 300

# Start/stop
pct start 300
pct stop 300

# Snapshot
pct snapshot 300 before-update
```

---

## 4. Windows Containers — How They Work

### The Core Architecture

Windows containers work on the same principle as Linux containers — they are isolated processes running on the **host Windows kernel** (`ntoskrnl.exe`). But the internals are entirely different because the Windows OS is fundamentally different from Linux.

A Windows container application makes system calls to:
- The **Windows Object Manager** (not Linux VFS)
- The **Windows Registry** (not `/etc` config files)
- **Win32 subsystem** or **Windows NT native API**
- **Windows networking stack** (Winsock, not POSIX sockets)

### Windows Container Base Images

Microsoft provides official base images. You choose based on what your app needs:

| Image | Size | Use Case |
|-------|------|----------|
| `mcr.microsoft.com/windows/nanoserver` | ~260–300MB | Smallest. .NET apps, CLI tools. cmd.exe only — no PowerShell unless installed separately. |
| `mcr.microsoft.com/windows/servercore` | ~3.5GB | Full Win32 API. PowerShell included. Most compatible. |
| `mcr.microsoft.com/windows/server` | ~5GB | Full Windows Server experience inside a container. |
| `mcr.microsoft.com/windows` | ~3GB | General purpose. Includes most APIs. |

> ⚠️ **Version pinning is mandatory.** Windows containers must match the host kernel version or use Hyper-V isolation. Example: `nanoserver:ltsc2022` runs on Windows Server 2022 or Windows 11 (22H2+) hosts. As of mid-2026, `ltsc2025` (Windows Server 2025) tags also exist as a current option — match whichever LTSC release your host actually runs. Mismatching without Hyper-V isolation causes immediate failure.

### The Two Isolation Modes (Critical Distinction)

#### Process Isolation

```
[ Windows Container A ]  [ Windows Container B ]
         │                        │
         ▼                        ▼
[ Host Windows Kernel (ntoskrnl.exe) ]  ← shared
         │
[ Physical Hardware ]
```

- Containers share the host Windows kernel directly
- Fastest performance, lowest overhead
- **Strict version requirement:** container OS version must match host OS version exactly
- Default mode on Windows Server

#### Hyper-V Isolation

```
[ Windows Container ]
         │
[ Micro-VM (lightweight Hyper-V partition) ]  ← per container
         │  has its own isolated kernel copy
[ Host Windows Kernel ]
         │
[ Physical Hardware ]
```

- Each container gets its own isolated, minimal Hyper-V VM
- Solves version mismatch in **standalone Docker** — the micro-VM provides the correct kernel version
- Slower startup (~1-2 seconds vs milliseconds), more overhead
- **Default mode on Windows 10/11 desktop** (because desktop kernel ≠ server kernel)
- Available as an option when running older container versions on newer Windows Server hosts **outside Kubernetes**

> ⚠️ **Windows 10/11 Desktop editions only support Hyper-V Isolation — Process Isolation is not available.** The kernel version difference between desktop editions and Windows Server editions makes direct process sharing impossible. This is not configurable. Only Windows Server 2019/2022/2025 supports Process Isolation.

> ⚠️ **Critical Kubernetes-specific exception: Hyper-V isolation is NOT supported inside Kubernetes at all.** This is easy to miss because Hyper-V isolation works perfectly well as a standalone `docker run --isolation=hyperv` escape hatch — but current Kubernetes documentation states plainly that Kubernetes does not support running Windows containers with Hyper-V isolation. Inside a Kubernetes cluster specifically, Process Isolation with an exact host/container OS version match is mandatory; there is no Hyper-V fallback for a version mismatch the way there is with plain Docker. Plan your Windows node OS version and your container base image version together from the start if Kubernetes is the target.

> ⚠️ **localhost port forwarding quirk on Windows desktop:** Because every Windows container on a desktop host runs inside its own Hyper-V micro-VM, `localhost:80` on the Windows host does not always reliably forward to a container port even when `-p 80:80` is specified. If a containerized web service appears to be running but the browser cannot reach `http://localhost:80`, grab the container's internal IP directly and hit that instead:
> ```powershell
> # Get the container's actual internal IP
> docker inspect <container-name> --format "{{.NetworkSettings.Networks.nat.IPAddress}}"
> # Then browse to that IP directly, e.g. http://172.24.48.3:80
> ```
> This is a known Docker Desktop on Windows desktop behavior — not a misconfiguration on your end. Windows Server with Process Isolation does not have this problem.

### Running Windows Containers (PowerShell)

```powershell
# Switch Docker to Windows container mode first (see Section 7)

# Pull a Windows base image
docker pull mcr.microsoft.com/windows/nanoserver:ltsc2022

# Run an interactive Windows container (cmd.exe shell)
docker run -it mcr.microsoft.com/windows/nanoserver:ltsc2022 cmd.exe

# Run with PowerShell (requires servercore, not nanoserver)
docker run -it mcr.microsoft.com/windows/servercore:ltsc2022 powershell.exe

# Run detached Windows container with port mapping
docker run -d `
  --name iis-web `
  -p 80:80 `
  mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# List running containers
docker ps

# Execute command inside running Windows container
docker exec -it iis-web cmd.exe

# Force Hyper-V isolation explicitly (standalone Docker only — not valid inside Kubernetes)
docker run -it --isolation=hyperv mcr.microsoft.com/windows/nanoserver:ltsc2019 cmd.exe

# Force process isolation explicitly
docker run -it --isolation=process mcr.microsoft.com/windows/nanoserver:ltsc2022 cmd.exe
```

### What the Windows Container Filesystem Looks Like

Inside a Windows container, the filesystem mirrors a standard Windows installation but is stripped down:

```
C:\
├── Program Files\
├── Program Files (x86)\
├── Users\
│   └── ContainerUser\     ← default non-admin user
├── Windows\
│   ├── System32\          ← core system binaries
│   └── SysWOW64\          ← 32-bit compatibility layer
└── (no D:\ or other drives unless mounted)
```

Environment variables inside a Windows container:

```cmd
# Inside container cmd.exe:
set                         # List all environment variables
echo %COMPUTERNAME%         # Container hostname
echo %OS%                   # Windows_NT
echo %PROCESSOR_ARCHITECTURE% # AMD64
ipconfig                    # Container's virtual network adapter
```

---

## 5. Windows VMs — How They Work

### Hyper-V Architecture

Windows 11 Pro and Windows Server include **Hyper-V**, a Type-1 hypervisor built directly into the OS. When Hyper-V is enabled, the Windows OS itself moves into a privileged VM called the **Root Partition**. All other VMs become **Child Partitions**.

```
[ Root Partition (your Windows 11 desktop) ]   [ Child VM: Ubuntu ]   [ Child VM: Windows Server ]
             │                                          │                       │
             └──────────────────────────────────────────┴───────────────────────┘
                                          │
                              [ Hyper-V Hypervisor ]  ← runs at ring -1 (below the OS)
                                          │
                              [ Physical Hardware ]
```

> ⚠️ **When Hyper-V is enabled, it takes control of hardware virtualization.** This means VirtualBox and VMware Workstation (Type-2 hypervisors) will either fail or run in slow emulation mode unless they are explicitly updated to support Hyper-V coexistence.

### Enabling Hyper-V

```powershell
# Enable all Hyper-V features
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -All

# Verify installation
Get-WindowsOptionalFeature -Online | Where-Object {$_.FeatureName -like "*Hyper-V*"}

# Alternative: DISM method
DISM /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
```

> ⚠️ Requires a full system reboot.

### Creating and Managing Hyper-V VMs (PowerShell)

```powershell
# Create a Generation 2 VM (UEFI, modern hardware)
New-VM `
  -Name "Ubuntu-Worker" `
  -MemoryStartupBytes 8GB `
  -Generation 2 `
  -NewVHDPath "C:\HyperV-Disks\Ubuntu-Worker.vhdx" `
  -NewVHDSizeBytes 100GB

# Attach installation ISO
Add-VMDvdDrive -VMName "Ubuntu-Worker" -Path "C:\ISOs\ubuntu-22.04-live-server-amd64.iso"

# Set boot order (DVD first for initial install)
$dvd = Get-VMDvdDrive -VMName "Ubuntu-Worker"
Set-VMFirmware -VMName "Ubuntu-Worker" -FirstBootDevice $dvd

# Set CPU and memory
Set-VM -Name "Ubuntu-Worker" -ProcessorCount 4
Set-VMMemory -VMName "Ubuntu-Worker" -DynamicMemoryEnabled $true -MinimumBytes 2GB -MaximumBytes 8GB

# Connect a virtual network switch
# First, create a switch if none exists:
New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Attach switch to VM
Connect-VMNetworkAdapter -VMName "Ubuntu-Worker" -SwitchName "ExternalSwitch"

# Start the VM
Start-VM -Name "Ubuntu-Worker"

# Open the VM console (Virtual Machine Connection)
vmconnect.exe localhost "Ubuntu-Worker"

# List all VMs and their state
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime

# Checkpoint (snapshot)
Checkpoint-VM -Name "Ubuntu-Worker" -SnapshotName "Pre-k3s"

# Restore checkpoint
Restore-VMCheckpoint -Name "Ubuntu-Worker" -VMCheckpointName "Pre-k3s" -Confirm:$false

# Export VM (for backup/migration)
Export-VM -Name "Ubuntu-Worker" -Path "D:\VM-Backups\"

# Import VM
Import-VM -Path "D:\VM-Backups\Ubuntu-Worker\Virtual Machines\*.vmcx"

# Remove VM completely
Remove-VM -Name "Ubuntu-Worker" -Force
```

### Hyper-V Virtual Switches

Virtual switches connect VMs to networks. There are three types:

| Switch Type | Connectivity | Use Case |
|-------------|-------------|----------|
| **External** | Bridged to a physical NIC — VM gets full LAN access | k3s worker nodes that need cluster network access |
| **Internal** | Host + VMs communicate, no external LAN | Isolated lab networks |
| **Private** | VMs only, host cannot see traffic | Completely isolated VM-to-VM networks |

```powershell
# List existing virtual switches
Get-VMSwitch

# Create External switch (bridges to physical NIC)
New-VMSwitch -Name "Lab-External" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Create Internal switch
New-VMSwitch -Name "Lab-Internal" -SwitchType Internal

# Create Private switch
New-VMSwitch -Name "Lab-Private" -SwitchType Private

# Attach VM to a switch
Connect-VMNetworkAdapter -VMName "Ubuntu-Worker" -SwitchName "Lab-External"
```

---

## 6. WSL2 — The Hybrid Bridge

### What WSL2 Actually Is

WSL2 is not an emulator or a compatibility layer. It is a **genuine Linux kernel** — compiled by Microsoft — running inside a lightweight, hidden Hyper-V utility VM that starts automatically when you open a WSL2 terminal.

```
[ Windows 11 Host ]
         │
[ Hyper-V Hypervisor ]  ← enabled automatically when WSL2 is installed
         │
[ Utility VM (hidden) ]
         │
[ Microsoft Linux Kernel (real kernel, real syscalls) ]
         │
[ Ubuntu 22.04 Userspace (your WSL2 distro) ]
         │
[ You: bash terminal ]
```

From the perspective of anything running inside WSL2 — including `k3s-agent` — it is running on a real Linux machine. It has real Linux kernel interfaces, real `/proc`, real cgroups, real network namespaces.

### WSL2 Filesystem Layout

```
# Inside WSL2 — standard Linux filesystem
/home/<user>/            ← your Linux home directory
/etc/                    ← Linux config files
/mnt/c/                  ← your Windows C:\ drive mounted here
/mnt/d/                  ← your Windows D:\ drive mounted here

# From Windows — access WSL2 filesystem via network path
\\wsl$\Ubuntu\           ← maps to WSL2 root /
\\wsl$\Ubuntu\home\<user>\ ← maps to /home/<user>/
```

> ⚠️ **Performance warning:** Accessing WSL2 files through `\\wsl$\` from Windows (or accessing Windows files from `/mnt/c/` inside WSL2) crosses the 9P filesystem bridge and is significantly slower than staying within one filesystem. Keep project files in the WSL2 native filesystem for performance.

### Configuring WSL2 Resource Limits

```powershell
# Create or edit the global WSL2 config
notepad "$env:USERPROFILE\.wslconfig"
```

```ini
[wsl2]
memory=48GB                  # Cap WSL2 total RAM (default: 80% of host)
processors=12                # Cap virtual CPU threads
swap=8GB                     # Virtual swap file size
swapFile=C:\\Temp\\wsl-swap.vhdx
localhostForwarding=true     # Forward WSL2 ports to Windows localhost
nestedVirtualization=true    # Allow VMs inside WSL2 (needed for some k3s configs)
kernelCommandLine=cgroup_no_v1=all  # Force cgroup v2 (required for some k3s versions)
```

```powershell
# Apply changes — must fully shut down WSL2
wsl --shutdown

# Verify shutdown
wsl --list --verbose

# Restart WSL2 (just open a new WSL2 terminal)
wsl

# Check resource limits inside WSL2
wsl -e free -h
wsl -e nproc
```

### WSL2 Distribution Management

```powershell
# List all installed WSL2 distros and their state
wsl --list --verbose

# Install a distro from Microsoft Store equivalent
wsl --install -d Ubuntu-22.04

# Set default distro
wsl --set-default Ubuntu-22.04

# Run a specific distro
wsl -d Ubuntu-22.04

# Export a distro (backup)
wsl --export Ubuntu-22.04 "D:\WSL-Backups\ubuntu-22-backup.tar"

# Import a distro (restore or clone)
wsl --import Ubuntu-Clone "C:\WSL\Ubuntu-Clone" "D:\WSL-Backups\ubuntu-22-backup.tar"

# Terminate a running distro
wsl --terminate Ubuntu-22.04

# Unregister (delete) a distro — DESTRUCTIVE
wsl --unregister Ubuntu-22.04

# Update the WSL2 kernel
wsl --update

# Check WSL version
wsl --version
```

---

## 7. Switching Between Linux and Windows Container Modes

### Why This Switch Exists

Docker Desktop on Windows runs **one daemon at a time** — either the Linux daemon (which runs containers inside a hidden WSL2 or Hyper-V Linux VM) or the Windows daemon (which runs native Windows containers against the host Windows kernel). They are mutually exclusive.

```
Docker Desktop
      │
      ├── Linux Mode ──→ [ Hidden Linux VM (WSL2 or Hyper-V) ] ──→ Linux containers
      │
      └── Windows Mode ──→ [ Host Windows Kernel ] ──→ Windows containers
```

### Switching Modes

```powershell
# Switch to Windows Container mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Switch back to Linux Container mode (same command — it toggles)
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Check which mode you are in
docker version
# Look for "OS/Arch" in Server section:
#   linux/amd64   → Linux mode
#   windows/amd64 → Windows mode

# Alternative check
docker info | findstr "OSType"
# OSType: linux   → Linux mode
# OSType: windows → Windows mode
```

> ⚠️ **The switch is abrupt.** All running containers in the previous mode become unreachable until you switch back. Your Docker Compose stacks in Linux mode will appear to be down while in Windows mode — they are not destroyed, just inaccessible. Switch back and they resume.

### The Docker Desktop Context System (Alternative)

Docker Desktop also supports contexts, which can be used to target different daemons:

```powershell
# List available contexts
docker context ls

# Create a context pointing to Windows daemon
docker context create windows-ctx --docker "host=npipe:////./pipe/docker_engine_windows"

# Use a specific context
docker context use windows-ctx

# Switch back to default (Linux)
docker context use default
```

### What Breaks During a Mode Switch

| Component | Linux Mode | Windows Mode |
|-----------|-----------|-------------|
| Your `docker-compose.yml` Linux stacks | ✅ Running | ❌ Inaccessible (daemon not active) |
| Windows container images | ❌ Won't run | ✅ Running |
| `docker ps` output | Linux containers | Windows containers |
| Volume mounts | Linux paths (`/home/...`) | Windows paths (`C:\...`) |
| Networking | Linux bridge (docker0) | Windows HNS (Host Network Service) |

---

## 8. The Windows Registry — What It Is and Why It Matters for Containers

### What the Registry Is

The Windows Registry is a hierarchical database that stores configuration settings for the Windows OS, hardware drivers, installed applications, and user preferences. Unlike Linux which stores config in plain text files under `/etc`, Windows centralizes configuration in the Registry.

Structure:

```
Registry Root Keys (Hives)
│
├── HKEY_LOCAL_MACHINE (HKLM)     ← system-wide settings, hardware, services
│   ├── SOFTWARE\                  ← installed application settings
│   ├── SYSTEM\                    ← device drivers, boot config, services
│   └── HARDWARE\                  ← detected hardware (rebuilt at boot)
│
├── HKEY_CURRENT_USER (HKCU)      ← current logged-in user settings
│   ├── SOFTWARE\                  ← per-user app settings
│   └── Environment\               ← user environment variables
│
├── HKEY_CLASSES_ROOT (HKCR)      ← file associations, COM objects
│
├── HKEY_USERS (HKU)              ← all user profiles
│
└── HKEY_CURRENT_CONFIG (HKCC)    ← current hardware profile
```

### Registry and Windows Containers

Every Windows container gets its **own isolated Registry hive** — separate from the host and from other containers. When a Windows container starts:

1. The base image provides a read-only Registry snapshot
2. A writable Registry layer is created for the container
3. The container's processes read/write to their isolated Registry
4. When the container stops, the writable layer is discarded (unless committed to an image)

This is the Registry equivalent of what OverlayFS does for the filesystem in Linux containers.

### Inspecting the Registry in a Running Windows Container

```powershell
# Enter a servercore container
docker run -it mcr.microsoft.com/windows/servercore:ltsc2022 powershell.exe

# Inside the container — Registry commands work normally:

# List subkeys
Get-ChildItem HKLM:\SOFTWARE

# Read a value
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name "ProductName"

# Set a value (persists only within container lifetime unless committed)
Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "InstallPath" -Value "C:\MyApp"

# Create a new key
New-Item -Path "HKLM:\SOFTWARE\MyApp" -Force

# Delete a key
Remove-Item -Path "HKLM:\SOFTWARE\MyApp" -Recurse

# Export a Registry key to a .reg file
reg export "HKLM\SOFTWARE\MyApp" C:\export.reg

# Import a .reg file
reg import C:\export.reg
```

### Registry Differences: Host vs Container

| Registry Location | Host Access | Container Access |
|-------------------|-------------|-----------------|
| `HKLM\SYSTEM\CurrentControlSet\Services` | Full host services | Container-specific services only |
| `HKLM\SOFTWARE` | Host applications | Container image applications |
| `HKCU` | Current Windows user | `ContainerUser` or `ContainerAdministrator` |
| Hardware keys | Physical hardware | Virtual/stub entries |

### PowerShell Registry Commands Reference

```powershell
# Navigate Registry like a filesystem
Set-Location HKLM:\SOFTWARE
Get-ChildItem                              # List keys (like ls)
Get-ItemProperty .                        # List values in current key

# Read specific value
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion").ProductName

# Check if key exists
Test-Path "HKLM:\SOFTWARE\MyApp"

# Get all values in a key
Get-Item "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" | Select-Object -ExpandProperty Property

# Search Registry (slow — searches all values)
Get-ChildItem -Path HKLM:\SOFTWARE -Recurse -ErrorAction SilentlyContinue |
  Where-Object { $_.Name -like "*Docker*" }

# Common Docker/container Registry paths on host:
# HKLM:\SYSTEM\CurrentControlSet\Services\docker        ← Docker daemon service config
# HKLM:\SOFTWARE\Docker Inc.\Docker Desktop             ← Docker Desktop settings
# HKLM:\SYSTEM\CurrentControlSet\Services\hns           ← Host Network Service
# HKLM:\SYSTEM\CurrentControlSet\Services\vmsmp         ← Hyper-V virtual machine bus
```

---

## 9. PowerShell Command Authority — Complete Reference

### Docker Commands in PowerShell

```powershell
# ───────── IMAGE MANAGEMENT ─────────
docker images                             # List local images
docker pull mcr.microsoft.com/windows/nanoserver:ltsc2022
docker rmi mcr.microsoft.com/windows/nanoserver:ltsc2022
docker image prune -a                     # Remove all unused images

# Build an image from a Dockerfile
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f Dockerfile.windows .

# ───────── CONTAINER LIFECYCLE ─────────
docker run -it <image> cmd.exe            # Interactive Windows shell
docker run -it <image> powershell.exe     # Interactive PowerShell
docker run -d --name myapp -p 80:80 <image>  # Detached
docker start myapp
docker stop myapp
docker restart myapp
docker rm myapp
docker rm -f myapp                        # Force remove running container

# ───────── INSPECTION ─────────
docker ps                                 # Running containers
docker ps -a                              # All containers (including stopped)
docker inspect myapp                      # Full JSON details
docker logs myapp
docker logs -f myapp                      # Follow log output
docker stats                              # Live resource usage
docker top myapp                          # Processes inside container

# ───────── NETWORKING ─────────
docker network ls
docker network inspect bridge
docker network create --driver nat mynet   # Windows uses 'nat', Linux uses 'bridge'
docker run --network mynet <image>

# ───────── VOLUMES ─────────
docker volume ls
docker volume create mydata
docker run -v mydata:C:\data <image>      # Windows path in container
docker run -v C:\host\path:C:\container\path <image>  # Bind mount

# ───────── SYSTEM ─────────
docker system prune                       # Remove stopped containers, unused networks, dangling images
docker system df                          # Disk usage breakdown
docker info                               # Full daemon info (OS type, version, storage driver)
```

### Hyper-V VM Management

```powershell
# ───────── VM LIFECYCLE ─────────
Get-VM                                    # List all VMs
Get-VM -Name "Ubuntu-Worker"             # Specific VM
Start-VM -Name "Ubuntu-Worker"
Stop-VM -Name "Ubuntu-Worker"            # Graceful shutdown
Stop-VM -Name "Ubuntu-Worker" -Force     # Hard stop (pull power)
Restart-VM -Name "Ubuntu-Worker"
Suspend-VM -Name "Ubuntu-Worker"         # Save state (hibernate)
Resume-VM -Name "Ubuntu-Worker"
Remove-VM -Name "Ubuntu-Worker" -Force

# ───────── VM CONFIGURATION ─────────
Set-VM -Name "Ubuntu-Worker" -ProcessorCount 8
Set-VMMemory -VMName "Ubuntu-Worker" -StartupBytes 8GB
Set-VMMemory -VMName "Ubuntu-Worker" -DynamicMemoryEnabled $true -MinimumBytes 2GB -MaximumBytes 16GB

# ───────── DISK MANAGEMENT ─────────
New-VHD -Path "C:\HyperV-Disks\data.vhdx" -SizeBytes 50GB -Dynamic
Add-VMHardDiskDrive -VMName "Ubuntu-Worker" -Path "C:\HyperV-Disks\data.vhdx"
Get-VMHardDiskDrive -VMName "Ubuntu-Worker"
Remove-VMHardDiskDrive -VMName "Ubuntu-Worker" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 1

# Compact a VHDX file (reclaim unused space)
Optimize-VHD -Path "C:\HyperV-Disks\Ubuntu-Worker.vhdx" -Mode Full

# ───────── SNAPSHOTS/CHECKPOINTS ─────────
Checkpoint-VM -Name "Ubuntu-Worker" -SnapshotName "Pre-k3s-install"
Get-VMCheckpoint -VMName "Ubuntu-Worker"
Restore-VMCheckpoint -VMName "Ubuntu-Worker" -VMCheckpointName "Pre-k3s-install" -Confirm:$false
Remove-VMCheckpoint -VMName "Ubuntu-Worker" -VMCheckpointName "Pre-k3s-install"

# ───────── NETWORKING ─────────
Get-VMSwitch
Get-VMNetworkAdapter -VMName "Ubuntu-Worker"
Add-VMNetworkAdapter -VMName "Ubuntu-Worker" -SwitchName "Lab-External"
Set-VMNetworkAdapter -VMName "Ubuntu-Worker" -StaticMacAddress "00-15-5D-00-01-01"

# ───────── MONITORING ─────────
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime
Measure-VM -Name "Ubuntu-Worker"         # Detailed resource measurement
Get-VMVideo -VMName "Ubuntu-Worker"      # Display adapter info
```

### WSL2 Management

```powershell
# ───────── DISTRO MANAGEMENT ─────────
wsl --list --verbose                      # List distros with state and WSL version
wsl --install -d Ubuntu-22.04            # Install distro
wsl --set-default Ubuntu-22.04           # Set default
wsl --set-version Ubuntu-22.04 2         # Upgrade distro to WSL2
wsl --shutdown                            # Stop ALL WSL2 distros and the utility VM
wsl --terminate Ubuntu-22.04             # Stop specific distro

# ───────── RUNNING COMMANDS ─────────
wsl                                       # Open default distro
wsl -d Ubuntu-22.04                       # Open specific distro
wsl -e ls /home                           # Run single command and exit
wsl -u root                               # Open as root
wsl -- cat /etc/os-release               # Run command with -- separator

# ───────── FILESYSTEM ─────────
# Access WSL2 filesystem from Windows File Explorer or PowerShell:
Get-ChildItem \\wsl$\Ubuntu\home\<user>\

# ───────── BACKUP AND RESTORE ─────────
wsl --export Ubuntu-22.04 "D:\Backups\ubuntu.tar"
wsl --import Ubuntu-Restored "C:\WSL\Ubuntu-Restored" "D:\Backups\ubuntu.tar"

# ───────── DIAGNOSTICS ─────────
wsl --version
wsl --status
wsl --update
```

### Windows Service Management (Relevant to Containers/Docker)

```powershell
# Docker daemon runs as a Windows service
Get-Service docker
Start-Service docker
Stop-Service docker
Restart-Service docker

# Host Network Service (HNS) — manages Windows container networking
Get-Service hns
Restart-Service hns                       # Fixes many Windows container networking issues

# ───────── DOCKER DAEMON CONFIG ─────────
# Windows Docker daemon config file location:
# C:\ProgramData\Docker\config\daemon.json

# Example daemon.json for Windows containers:
$config = @"
{
  "storage-engine": "windowsfilter",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "C:\\ProgramData\\Docker"
}
"@
$config | Set-Content "C:\ProgramData\Docker\config\daemon.json"

# Restart Docker after config change
Restart-Service docker
```

---

## 10. Standalone Usage — Running Without Kubernetes

### Linux Containers Standalone

Covered in Section 2. Key tools: `docker`, `docker compose`, `podman`, `nerdctl`.

Proxmox LXC containers are always standalone (they do not run inside Kubernetes — Kubernetes runs inside them or alongside them).

### Windows Containers Standalone

```powershell
# Switch to Windows container mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Verify mode
docker info | findstr "OSType"

# Run IIS web server (Windows-only)
docker run -d -p 80:80 --name iis mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# Run a .NET Framework app (Windows-only)
docker run -d mcr.microsoft.com/dotnet/framework/aspnet:4.8

# Run SQL Server Express on Windows containers
docker run -d `
  -e "ACCEPT_EULA=Y" `
  -e "SA_PASSWORD=YourPassword123!" `
  -p 1433:1433 `
  mcr.microsoft.com/mssql/server:2022-latest

# Windows container with volume mount
docker run -d `
  -v C:\AppData:C:\data `
  --name mywinapp `
  mywinapp:latest
```

### Docker Compose for Windows Containers

```yaml
# docker-compose.windows.yml
services:
  iis:
    image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
    ports:
      - "80:80"
    volumes:
      - C:\inetpub\wwwroot:C:\inetpub\wwwroot
    isolation: process          # or hyperv

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1433:1433"
    volumes:
      - sqldata:C:\data

volumes:
  sqldata:
```

```powershell
docker compose -f docker-compose.windows.yml up -d
```

---

## 11. Kubernetes Integration — Linux Containers

### How Kubernetes Runs Linux Containers

Kubernetes does not use Docker directly. It uses a **Container Runtime Interface (CRI)**-compliant runtime:

| Runtime | Used By | Notes |
|---------|---------|-------|
| `containerd` | k3s (default), most production clusters | Lightweight, no Docker daemon required |
| `CRI-O` | OpenShift | RedHat's runtime |
| `Docker` (via dockershim) | Legacy — removed in k8s 1.24+ | No longer supported |

k3s ships with `containerd` built in. When Kubernetes schedules a Pod, it calls `containerd`, which pulls the image and creates the container.

### Pod = One or More Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitea-pod
  namespace: infrastructure
spec:
  containers:
    - name: gitea
      image: gitea/gitea:latest
      ports:
        - containerPort: 3000
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "1000m"
      volumeMounts:
        - name: gitea-data
          mountPath: /data
  volumes:
    - name: gitea-data
      persistentVolumeClaim:
        claimName: gitea-pvc
```

### Kubernetes Node Requirements for Linux Containers

- Node OS: Any Linux distribution (Ubuntu, Debian, RHEL, Alpine)
- Container runtime: containerd, CRI-O
- Kernel: 4.15+ recommended, 5.x+ for full cgroup v2 support
- Automatic label applied by Kubernetes: `kubernetes.io/os: linux`

### Deploying to Linux Nodes in a Mixed Cluster

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: app
          image: nginx:latest
```

---

## 12. Kubernetes Integration — Windows Containers

> 🚨 **HARD ARCHITECTURAL RULE: Windows nodes can ONLY ever be Worker nodes.** etcd, the Kubernetes API server, the scheduler, and the controller manager — all control plane components — cannot run on Windows. This is not a configuration limitation or a temporary gap. The Kubernetes control plane is Linux-only, permanently. Any Windows machine joining your cluster joins exclusively as a Worker.

### Requirements for Windows Worker Nodes

- **Windows Server 2022 or Windows Server 2025** for current Kubernetes releases (Windows Server 2019 was supported in older Kubernetes versions but is no longer listed as a tested/supported Windows Server release for current Kubernetes — check the Kubernetes Windows OS version compatibility page for your specific Kubernetes version before standardizing on an OS image)
- Desktop editions (Windows 10/11) are not supported as production Kubernetes nodes
- WSL2-based Windows machines **can** join as Linux worker nodes (they expose a Linux kernel to k3s)
- The **Control Plane must always be Linux** — no exceptions
- Container runtime on Windows nodes: `containerd` with the `containerd-shim-runhcs-v1` shim (handles Windows containers)
- **Kubernetes does not support Hyper-V isolation for Windows containers at all** — only Process Isolation, which requires the Windows node's OS build to exactly match the container's base image version (see Section 4's isolation-mode discussion)

> ⚠️ **k3s's Windows agent support is less mature than standard Kubernetes' Windows worker support.** Standard Kubernetes (via `kubeadm`) has well-established, documented Windows worker node support going back several releases. k3s's own project maintainers have described Windows node support as still experimental and under active development. If a production-grade mixed Linux/Windows cluster is the goal, evaluate whether standard `kubeadm`-based Kubernetes is a better fit for the Windows side than k3s — or budget real troubleshooting time if proceeding with k3s's Windows agent, rather than expecting the same plug-and-play experience as joining a Linux agent.

### Joining a Windows Node to k3s

```powershell
# On the Windows Server node — run in PowerShell as Administrator

# Install k3s agent for Windows
Invoke-WebRequest -Uri "https://github.com/k3s-io/k3s/releases/latest/download/k3s-windows-amd64.exe" -OutFile "k3s.exe"

# Install as a Windows service
.\k3s.exe agent `
  --server "https://100.64.10.5:6443" `
  --token "K10a76be04c538a7299042b938f30713bc4928374a8::server:secrettoken" `
  --node-ip "100.64.10.20"

# Verify the node appears in the cluster (run from Linux master)
kubectl get nodes -o wide
```

### Deploying Windows Containers in Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-iis
  namespace: windows-workloads
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iis
  template:
    metadata:
      labels:
        app: iis
    spec:
      nodeSelector:
        kubernetes.io/os: windows          # MANDATORY — must target Windows node
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"
      containers:
        - name: iis
          image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
          ports:
            - containerPort: 80
          resources:
            limits:
              memory: "2Gi"
              cpu: "2"
```

> ⚠️ Windows nodes in k3s often require a **taint** applied to prevent Linux pods from accidentally scheduling there. Apply the taint and then add the matching toleration to Windows workloads.

```bash
# Taint the Windows node
kubectl taint nodes <windows-node-name> os=windows:NoSchedule

# Verify
kubectl describe node <windows-node-name> | grep Taint
```

### Windows Container Limitations in Kubernetes

| Feature | Linux Pods | Windows Pods |
|---------|-----------|-------------|
| Host networking | ✅ | ⚠️ Limited |
| Privileged containers | ✅ | ❌ Not supported |
| HostPath volumes | ✅ | ✅ (Windows paths) |
| Persistent volumes (RWX) | ✅ | ⚠️ Limited |
| Init containers | ✅ | ✅ |
| DaemonSets | ✅ | ✅ |
| StatefulSets | ✅ | ✅ |
| GPU scheduling | ✅ | ✅ (DirectX GPU) |
| Linux system calls (seccomp) | ✅ | ❌ |

---

## 13. Kubernetes Integration — VMs (KubeVirt)

### What KubeVirt Is

KubeVirt is a Kubernetes extension that allows you to run full **Virtual Machines** as Kubernetes resources — declared in YAML, scheduled like pods, managed with `kubectl`. The VMs run under KVM but are orchestrated by Kubernetes.

This means you can run a Windows Server VM as a Kubernetes workload — not as a Windows container, but as a full VM with its own kernel.

### When to Use KubeVirt vs Windows Containers

| Scenario | Use KubeVirt VM | Use Windows Container |
|----------|----------------|----------------------|
| Legacy Windows app that needs full OS | ✅ | ❌ |
| App requiring specific Windows kernel version | ✅ | ⚠️ (Hyper-V isolation, standalone Docker only — not available inside Kubernetes) |
| High-density microservices | ❌ | ✅ |
| App using kernel-mode drivers | ✅ | ❌ |
| Fast startup required | ❌ | ✅ |

### Basic KubeVirt VM Manifest

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: windows-server-vm
  namespace: vms
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 4
        memory:
          guest: 8Gi
        devices:
          disks:
            - name: os-disk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: os-disk
          persistentVolumeClaim:
            claimName: windows-server-pvc
```

---

## 14. Cross-Platform Connectivity — How Linux and Windows Nodes Communicate

### The Network Layer

In a hybrid Kubernetes cluster, Linux and Windows nodes communicate over standard IP networking. The CNI (Container Network Interface) plugin handles pod-to-pod communication across OS boundaries.

**k3s default CNI: Flannel**

Flannel creates an overlay network (VXLAN by default) that encapsulates pod traffic across nodes. Both Linux and Windows pods get an IP from the pod CIDR range. A Linux pod can reach a Windows pod and vice versa — the OS boundary is transparent at the network layer.

```
[ Linux Pod: 10.42.0.5 ]  ──────── VXLAN overlay ────────  [ Windows Pod: 10.42.1.8 ]
         │                                                      │
[ Linux Node: 100.64.10.5 ]  ── Tailscale WireGuard ──  [ Windows Node: 100.64.10.20 ]
```

### Tailscale for Hybrid Clusters

```bash
# On Linux nodes — install and auth Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --authkey=<your-key>

# Get Tailscale IP
tailscale ip -4
```

```powershell
# On Windows nodes — install Tailscale
winget install Tailscale.Tailscale

# Authenticate (opens browser)
tailscale up

# Get Tailscale IP
tailscale ip -4

# Verify connectivity to Linux master
ping 100.64.10.5
```

### Flannel VXLAN Port — Critical Windows Firewall Rule (and it's NOT the same port as Linux)

Flannel's VXLAN overlay uses different default UDP ports depending on the node's OS — this is a detail worth getting exactly right, because the two numbers look similar enough to mix up:

| OS | Default Flannel VXLAN UDP port |
|---|---|
| Linux | **8472** |
| Windows | **4789** |

This split is intentional and documented by the Flannel project itself — Windows VXLAN support uses a different port than the Linux kernel's default. On Linux nodes this port is handled automatically by the CNI setup. On Windows nodes, the Windows Firewall blocks UDP 4789 by default, which causes pod-to-pod communication across OS boundaries to silently fail — pods appear healthy but cannot reach each other across the Linux/Windows node boundary.

You must explicitly allow UDP 4789 on the Tailscale adapter on every Windows node joining the cluster, **and** separately ensure UDP 8472 is reachable between Linux nodes (typically open by default on a homelab LAN/Tailscale mesh, but worth verifying explicitly if a host firewall is active on the Linux side too):

```powershell
# Allow Flannel VXLAN traffic inbound on the Tailscale adapter (Windows node, port 4789)
New-NetFirewallRule `
  -Name "Flannel-VXLAN-Inbound" `
  -DisplayName "Flannel VXLAN Overlay (Inbound)" `
  -Protocol UDP `
  -LocalPort 4789 `
  -Direction Inbound `
  -Action Allow `
  -InterfaceAlias "Tailscale"

# Allow outbound as well
New-NetFirewallRule `
  -Name "Flannel-VXLAN-Outbound" `
  -DisplayName "Flannel VXLAN Overlay (Outbound)" `
  -Protocol UDP `
  -LocalPort 4789 `
  -Direction Outbound `
  -Action Allow `
  -InterfaceAlias "Tailscale"

# Verify the rules were created
Get-NetFirewallRule | Where-Object { $_.Name -like "Flannel*" }
```

```bash
# On Linux nodes, if a host firewall (ufw/firewalld) is active, allow UDP 8472
# from the trusted node subnet — replace <node-subnet> with your actual range
ufw allow proto udp from <node-subnet> to any port 8472
# or, firewalld:
firewall-cmd --permanent --zone=trusted --add-port=8472/udp
firewall-cmd --reload
```

> ⚠️ **Symptom if either side's port is missing:** Linux pods can reach other Linux pods across nodes fine. Windows pods can reach other Windows pods on the same node fine. But Linux pod → Windows pod (or vice versa) across node boundaries times out silently. `kubectl exec` into a Linux pod and `curl` the Windows service IP will hang indefinitely. Check both port numbers — 8472 on the Linux side, 4789 on the Windows side — rather than assuming a single port number covers the whole cluster.

### Service Communication Across OS Boundaries

A Linux pod can call a Windows pod's service using standard Kubernetes DNS:

```
http://windows-iis-service.windows-workloads.svc.cluster.local:80
```

This works exactly the same as Linux-to-Linux service calls. Kubernetes DNS (CoreDNS) resolves the name to the ClusterIP, kube-proxy forwards to the pod IP, Flannel routes across nodes.

### Windows Host Network Service (HNS)

On Windows nodes, the **Host Network Service (HNS)** is the Windows equivalent of Linux's `iptables`/`netfilter`. It manages:

- Windows container networking endpoints
- NAT rules for container port mapping
- Virtual switch assignments
- Kubernetes service routing on Windows nodes

```powershell
# Inspect HNS networks (Windows containers' network view)
Get-HnsNetwork

# Inspect HNS endpoints (individual container network connections)
Get-HnsEndpoint

# Restart HNS (fixes most Windows container networking issues)
Restart-Service hns

# View HNS policies (equivalent to iptables rules)
Get-HnsPolicyList
```

---

## 15. Scheduling and Affinity — Keeping Workloads on the Right OS

### Node Selectors (Simple)

```yaml
spec:
  nodeSelector:
    kubernetes.io/os: linux      # or: windows
    kubernetes.io/arch: amd64
```

### Node Affinity (Advanced)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                  - linux
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - local-datacenter
```

### Taints and Tolerations

Taints on a node **repel** pods. Tolerations on a pod **allow** it to schedule on a tainted node.

```bash
# Taint a Windows node to prevent Linux pods
kubectl taint nodes win-node-1 os=windows:NoSchedule

# Taint a GPU node to reserve it
kubectl taint nodes gpu-node-1 nvidia.com/gpu=present:NoSchedule

# Remove a taint
kubectl taint nodes win-node-1 os=windows:NoSchedule-
```

```yaml
# Pod toleration to schedule on Windows node
spec:
  tolerations:
    - key: "os"
      operator: "Equal"
      value: "windows"
      effect: "NoSchedule"
```

### Custom Node Labels for Your Homelab

```bash
# Label nodes by location
kubectl label node lab-1 homelab/location=local-rack
kubectl label node lab-6 homelab/location=local-rack
kubectl label node remote-edge homelab/location=remote-edge

# Label by hardware tier
kubectl label node beelink homelab/tier=high-compute
kubectl label node optiplex-1 homelab/tier=worker

# View all labels on a node
kubectl get node lab-1 --show-labels

# Select pods by custom label
kubectl get pods -l homelab/tier=high-compute
```

---

## 16. Storage Boundaries Across OS Types

### Linux Container Storage

```
Linux Container
      │
      ├── Image Layers (read-only, OverlayFS)
      │     └── /usr, /lib, /bin from image
      │
      └── Writable Layer (ephemeral — lost on container removal)
            └── Volume Mount (/data → PVC or hostPath)
```

Volume types available to Linux pods:
- `emptyDir` — temporary, lives with pod
- `hostPath` — mounts host filesystem path
- `persistentVolumeClaim` — Longhorn, NFS, local-path
- `configMap` / `secret` — config injection

### Windows Container Storage

Windows containers use **Windows Filter (windowsfilter)** instead of OverlayFS.

```
Windows Container
      │
      ├── Image Layers (read-only, windowsfilter)
      │     └── C:\Windows, C:\Program Files from image
      │
      └── Writable Layer (ephemeral)
            └── Volume Mount (C:\data → hostPath or PVC)
```

> ⚠️ **Windows path format in volumes:**

```yaml
# Linux pod volume mount
volumeMounts:
  - mountPath: /data        # Linux path format

# Windows pod volume mount
volumeMounts:
  - mountPath: C:\data      # Windows path format — backslash, drive letter
```

### The WSL2 Disk File

WSL2 stores its entire Linux filesystem inside a single `.vhdx` file on the Windows host:

```
C:\Users\<User>\AppData\Local\Packages\CanonicalGroupLimited...\LocalState\ext4.vhdx
```

This file grows dynamically as data is written but **does not shrink automatically** when data is deleted.

```powershell
# Compact the WSL2 disk file (reclaim space after large deletions)
# Step 1: Shut down WSL2
wsl --shutdown

# Step 2: Open diskpart
diskpart

# Step 3: In diskpart:
select vdisk file="C:\Users\<User>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

---

## 17. Networking Differences Between Linux and Windows Containers

### Linux Container Networking

Linux containers use `iptables`/`nftables` for NAT and port forwarding. Docker creates a virtual bridge (`docker0`). Each container gets a `veth` (virtual ethernet) pair — one end in the container's network namespace, one end attached to the bridge.

```
[ Container A: eth0 172.17.0.2 ]   [ Container B: eth0 172.17.0.3 ]
         │ veth pair                         │ veth pair
         ▼                                   ▼
[ docker0 bridge: 172.17.0.1 ]
         │
[ iptables NAT rules ]
         │
[ Host NIC: 192.168.1.100 ]
         │
[ External Network ]
```

### Windows Container Networking

Windows containers use **HNS (Host Network Service)** and **VFP (Virtual Filtering Platform)** — the Windows equivalents of `iptables` and Linux bridges. Each Windows container gets a virtual NIC attached to a Hyper-V virtual switch.

```
[ Container A ]   [ Container B ]
      │                 │
[ HNS Endpoint ]  [ HNS Endpoint ]
      │                 │
[ Hyper-V Virtual Switch (HNS Network) ]
      │
[ WinNAT / VFP rules ]
      │
[ Host NIC ]
      │
[ External Network ]
```

### Windows Container Network Drivers

| Driver | Purpose | Notes |
|--------|---------|-------|
| `nat` | Default — containers get private IPs, NAT to host | Equivalent to Linux `bridge` |
| `transparent` | Containers get IPs from physical network (like bridge mode) | Requires managed switch support |
| `overlay` | Multi-host networking (Swarm/Kubernetes) | Used by Flannel on Windows |
| `l2bridge` | Direct Layer 2 connectivity | Enterprise use |

```powershell
# Create a NAT network (Windows default)
docker network create --driver nat mynatnet

# Create a transparent network
docker network create --driver transparent mytransnet

# List networks
docker network ls

# Inspect a network
docker network inspect nat

# HNS-level network inspection
Get-HnsNetwork | Select-Object Name, Type, AddressPrefix
```

---

## 18. Troubleshooting Cross-Platform Failures

### Linux Container Failures

```bash
# Pod stuck in Pending
kubectl describe pod <pod-name>           # Look at Events section for scheduling failures
kubectl get events --sort-by='.lastTimestamp'

# Pod in CrashLoopBackOff
kubectl logs <pod-name>                   # Current logs
kubectl logs <pod-name> --previous        # Logs from last crash

# Pod in ImagePullBackOff
kubectl describe pod <pod-name>           # Check image name and registry auth
docker pull <image>                       # Test image pull directly on node

# Node NotReady
kubectl describe node <node-name>         # Check conditions
journalctl -u k3s -f                      # k3s agent logs on the node
systemctl status k3s-agent                # Agent service status
```

### Windows Container Failures

```powershell
# Container fails to start — check logs
docker logs <container-name>

# Version mismatch error
# Error: "the container operating system does not match the host operating system"
# Fix (standalone Docker only): Use Hyper-V isolation
docker run --isolation=hyperv <image>
# Fix (inside Kubernetes): rebuild/retag with a base image matching the
# Windows node's exact OS version -- Hyper-V isolation is not an option here

# HNS networking failure
# Error: "HNS failed with error: The object already exists"
# Fix:
Restart-Service hns
Restart-Service docker

# Clean up HNS state (nuclear option — fixes persistent network corruption)
Stop-Service docker
Stop-Service hns
Get-HnsNetwork | Remove-HnsNetwork         # Removes all container networks
Start-Service hns
Start-Service docker

# Container exits immediately — check what process is running
docker run -it <image> cmd.exe             # Replace your entrypoint with a shell

# Image pull fails on Windows
# Ensure you are in Windows container mode
docker info | findstr "OSType"

# WSL2 integration broken in Docker Desktop
wsl --shutdown
# Restart Docker Desktop

# Disk space exhaustion from container layers
docker system prune -a                    # Remove all unused images and containers
```

### Kubernetes Mixed-OS Failures

```bash
# Linux pod scheduled on Windows node (wrong OS crash)
kubectl get pod <pod-name> -o yaml | grep nodeName
kubectl describe pod <pod-name>
# Fix: Add nodeSelector kubernetes.io/os: linux to deployment

# Windows pod won't schedule — node not found
kubectl get nodes --show-labels | grep windows
# Check Windows node has label kubernetes.io/os: windows
# Check taint/toleration match

# Cross-OS service communication failing
kubectl exec -it <linux-pod> -- curl http://windows-service:80
# If DNS fails: kubectl exec -it <linux-pod> -- nslookup windows-service
# If connection refused: check Windows container port mapping and HNS rules

# Flannel VXLAN not bridging between OS types
kubectl get pods -n kube-system | grep flannel
kubectl logs -n kube-system <flannel-pod>
# On Windows node — check flannel service
Get-Service flanneld
# Remember: Linux side uses UDP 8472, Windows side uses UDP 4789 -- check
# firewall rules for BOTH numbers, on both node types, before assuming the
# CNI itself is broken
```

---

## 19. Architectural Gotchas and Hard Rules

1. **Control Plane is Linux-only, always.** etcd cannot run on Windows. The Kubernetes API server cannot run on Windows. This is non-negotiable and will not change.

2. **Container images are not cross-platform.** A container image built on Linux (e.g., `nginx:latest`) is compiled for Linux/amd64. It cannot run on a Windows kernel, period. There is no compatibility shim. Windows containers must use Windows base images from `mcr.microsoft.com`.

3. **Multi-platform images exist but are automatic.** Docker Hub images like `nginx:latest` are actually image manifests that point to different actual images per OS/arch. When you `docker pull nginx:latest` on Linux you get the Linux amd64 image. On Windows you might get a Windows image if one exists — but most open-source images only have Linux variants.

4. **Hyper-V isolation has overhead, and isn't available everywhere.** Every container in Hyper-V isolation mode boots a micro-VM. On a system running 50 containers, that is 50 micro-VMs. Memory overhead is real. Process isolation is faster but requires exact version matching. And inside Kubernetes specifically, Hyper-V isolation isn't supported at all — Process Isolation with exact version matching is the only option there.

5. **WSL2 is a Linux worker, not a Windows worker.** When you run `k3s-agent` inside WSL2, the Kubernetes cluster sees a Linux node, not a Windows node. Windows containers cannot run on a WSL2-based worker. WSL2 runs Linux containers only.

6. **Docker Desktop mode switching kills your active stacks temporarily.** Switching from Linux to Windows mode does not destroy containers, but makes them unreachable. Plan maintenance windows for mode switches if any Linux stacks are serving traffic.

7. **The WSL2 ext4.vhdx grows but never shrinks automatically.** After heavy container usage, compact it periodically with diskpart.

8. **Windows container networking resets on HNS restart.** All container network connections drop when you restart HNS. Running `docker ps` after will show containers are still alive but their IPs may have shifted. Restart containers after HNS restart.

9. **Windows Kubernetes nodes need the Flannel Windows overlay, and a different VXLAN port than Linux.** The standard Linux Flannel DaemonSet does not run on Windows. k3s handles this automatically, but on raw Kubernetes you must deploy the Windows-specific Flannel manifest separately — and remember that Flannel's VXLAN default port is 4789 on Windows, not the 8472 used on Linux.

10. **etcd latency kills hybrid clusters at WAN distances.** Keep all control plane nodes under 10ms RTT from each other. Remote Windows or Linux workers are fine as Workers over Tailscale. etcd over WAN = cluster freeze.

11. **k3s's Windows worker support is less mature than standard Kubernetes' Windows worker support.** Treat it as an evolving feature rather than a fully battle-tested one — budget extra troubleshooting time, and check the k3s project's current issue tracker for known Windows-agent limitations before committing a production workload to it.

---

## 20. Homelab Practice Tasks

### Task 1: Verify Your Container Mode and Run Both Types

```powershell
# Step 1: Check current mode
docker info | findstr "OSType"

# Step 2: Run a Linux container (must be in Linux mode)
docker run --rm alpine:latest echo "Linux container running"

# Step 3: Switch to Windows mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
docker info | findstr "OSType"

# Step 4: Run a Windows container
docker run --rm mcr.microsoft.com/windows/nanoserver:ltsc2022 cmd /c "echo Windows container running"

# Step 5: Switch back to Linux mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
```

### Task 2: Explore the Windows Registry Inside a Container

```powershell
# Run a servercore container with PowerShell
docker run -it mcr.microsoft.com/windows/servercore:ltsc2022 powershell.exe

# Inside the container:
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" | Select-Object ProductName, CurrentBuild
Get-ChildItem HKLM:\SYSTEM\CurrentControlSet\Services | Select-Object -First 10 Name
New-Item -Path "HKLM:\SOFTWARE\HomelabTest" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\HomelabTest" -Name "Version" -Value "1.0"
Get-ItemProperty "HKLM:\SOFTWARE\HomelabTest"
exit

# After exiting — verify registry change is gone (new container = fresh registry)
docker run -it mcr.microsoft.com/windows/servercore:ltsc2022 powershell.exe
Test-Path "HKLM:\SOFTWARE\HomelabTest"   # Should return False
exit
```

### Task 3: Create and Manage a Hyper-V VM

```powershell
# Create a VM
New-VM -Name "LabTest-VM" -MemoryStartupBytes 2GB -Generation 2 -NewVHDPath "C:\HyperV-Disks\LabTest.vhdx" -NewVHDSizeBytes 20GB

# List it
Get-VM

# Take a checkpoint
Checkpoint-VM -Name "LabTest-VM" -SnapshotName "Initial"

# Modify something (change memory)
Set-VMMemory -VMName "LabTest-VM" -StartupBytes 4GB

# Roll back
Restore-VMCheckpoint -Name "LabTest-VM" -VMCheckpointName "Initial" -Confirm:$false

# Verify memory rolled back
Get-VMMemory -VMName "LabTest-VM"

# Clean up
Remove-VM -Name "LabTest-VM" -Force
Remove-Item "C:\HyperV-Disks\LabTest.vhdx"
```

### Task 4: Verify Kubernetes Node OS Labels in Your Cluster

```bash
# From your Linux master node:
kubectl get nodes --show-labels

# Check the OS label specifically
kubectl get nodes -L kubernetes.io/os

# Apply a custom homelab label
kubectl label node <node-name> homelab/role=storage-node

# Verify
kubectl get node <node-name> --show-labels | tr ',' '\n' | grep homelab

# Deploy a test pod constrained to Linux nodes only
kubectl run linux-only-test \
  --image=nginx:latest \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/os":"linux"}}}' \
  --restart=Never

kubectl get pod linux-only-test -o wide   # Verify it landed on a Linux node
kubectl delete pod linux-only-test
```

### Task 5: WSL2 Resource Verification

```powershell
# Set a memory cap
$config = @"
[wsl2]
memory=8GB
processors=4
"@
$config | Set-Content "$env:USERPROFILE\.wslconfig"

# Restart WSL2
wsl --shutdown
Start-Sleep -Seconds 3

# Verify inside WSL2
wsl -e free -h
wsl -e nproc

# Clean up — remove the cap
Remove-Item "$env:USERPROFILE\.wslconfig"
wsl --shutdown
```

---

## Quick Reference Tables

### Container vs VM Decision Matrix

| Need | Use |
|------|-----|
| Run Linux app, fast startup, low overhead | Linux container (Docker/k3s pod) |
| Run Windows .NET/IIS app in Kubernetes | Windows container pod |
| Run legacy Windows app needing full OS | Hyper-V VM or KubeVirt VM |
| Full isolation between workloads | VM (any hypervisor) |
| High density (100+ instances per host) | Containers |
| Proxmox full service instance | LXC container or KVM VM |
| Homelab service (Gitea, Mattermost) | Proxmox LXC or Docker Compose stack |

### Mode Switch Quick Reference

| Goal | Command |
|------|---------|
| Check current Docker mode | `docker info \| findstr "OSType"` |
| Switch Docker daemon mode | `& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon` |
| Shut down all WSL2 instances | `wsl --shutdown` |
| List WSL2 distros and state | `wsl --list --verbose` |
| Enter default WSL2 distro | `wsl` |
| Restart Docker service | `Restart-Service docker` |
| Fix Windows container networking | `Restart-Service hns` |

### Kubernetes OS Targeting Quick Reference

| Goal | YAML |
|------|------|
| Force Linux node | `nodeSelector: kubernetes.io/os: linux` |
| Force Windows node | `nodeSelector: kubernetes.io/os: windows` |
| Force local zone | `nodeSelector: topology.kubernetes.io/zone: local-datacenter` |
| Taint Windows node | `kubectl taint nodes <name> os=windows:NoSchedule` |
| Tolerate Windows taint | `tolerations: [{key: "os", value: "windows", effect: "NoSchedule"}]` |

### Flannel VXLAN Port Quick Reference

| OS | Default UDP Port |
|---|---|
| Linux | 8472 |
| Windows | 4789 |

### File and Path Reference

| Path | Purpose |
|------|---------|
| `C:\Users\<User>\.wslconfig` | WSL2 global resource limits |
| `C:\ProgramData\Docker\config\daemon.json` | Windows Docker daemon config |
| `\\wsl$\Ubuntu\` | WSL2 filesystem from Windows |
| `/mnt/c/` | Windows C:\ from inside WSL2 |
| `/etc/rancher/k3s/k3s.yaml` | k3s cluster kubeconfig |
| `/var/lib/rancher/k3s/storage/` | k3s local-path storage |
| `C:\Users\<User>\AppData\Local\Packages\...\LocalState\ext4.vhdx` | WSL2 Linux disk file |
| `C:\ProgramData\Docker\` | Docker root data directory (Windows) |
| `HKLM:\SYSTEM\CurrentControlSet\Services\docker` | Docker Windows service Registry entry |
| `HKLM:\SYSTEM\CurrentControlSet\Services\hns` | Host Network Service Registry entry |

---

*File: `windows-linux-containers-vms-complete.md`*
*See also: `kubernetes-hybrid-architecture.md` | `windows-dockerfile-patterns.md`*
