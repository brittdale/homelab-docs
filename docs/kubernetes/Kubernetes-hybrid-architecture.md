# Kubernetes Hybrid Architecture — Platforms, Subsystems, and Containers

**Domain:** Advanced Cluster Orchestration & Architecture (GitOps Workflow)  
**Exam Competency:** Configure hybrid OS worker nodes, understand container kernel abstractions, and manage multi-platform storage boundaries.  
**Last Updated:** 2026-06-13  
**File Path:** `Kubernetes-hybrid-architecture.md`

---

## Summary

Kubernetes abstracts infrastructure by serving as a declarative API layer above the underlying operating system. At the enterprise and homelab level, this enables **Heterogeneous and Hybrid OS Clusters** — where physical Linux nodes, Windows desktops utilizing WSL2, and native Windows Servers operate within the same cluster fabric.

Coordinating this multi-platform landscape requires understanding how container processes interact directly with host kernels, managing the strict low-latency requirements of the distributed cluster control plane database (etcd), and routing cross-platform payloads safely over an encrypted **Tailscale Mesh WireGuard Network**.

> ⚠️ **ARCHITECT FOCUS:** While Kubernetes provides total infrastructure abstraction, it cannot break the laws of physics or operating system design. The cluster **Control Plane (Master nodes) must run on a native Linux distribution.** Windows nodes can *only* act as Worker nodes. Native Windows worker nodes cannot run Linux binary containers — they run Windows containers that interface directly with the Windows kernel (`ntoskrnl.exe`).

---

## Table of Contents

1. [Architectural Layering — The Cross-Platform Blueprint](#1-architectural-layering--the-cross-platform-blueprint)
2. [The Kernel Split — Linux Containers vs. Windows Containers](#2-the-kernel-split--linux-containers-vs-windows-containers)
3. [Deep Dive: Microsoft Virtualization & Isolation Modes](#3-deep-dive-microsoft-virtualization--isolation-modes)
4. [Geographic Boundaries & Latency: The Remote Worker Rule](#4-geographic-boundaries--latency-the-remote-worker-rule)
5. [Tailscale Mesh Network Topology](#5-tailscale-mesh-network-topology)
6. [Node Governance — Labels, Selectors, and Affinity](#6-node-governance--labels-selectors-and-affinity)
7. [The Windows Command Center — PowerShell Storage & Execution](#7-the-windows-command-center--powershell-storage--execution)
8. [Key Files, Paths, and Endpoints](#8-key-files-paths-and-endpoints)
9. [Worked Examples](#9-worked-examples)
10. [Troubleshooting Multi-OS Scenarios](#10-troubleshooting-multi-os-scenarios)
11. [Architectural Gotchas](#11-architectural-gotchas)
12. [Homelab Practice Tasks](#12-homelab-practice-tasks)
- [Quick Reference](#quick-reference)

---

## 1. Architectural Layering — The Cross-Platform Blueprint

When a command is issued to a hybrid Kubernetes cluster, it flows through clear abstraction barriers. The configuration state is declared in a single source of truth (**Gitea**), reconciled by an engine (**ArgoCD**), and executed across diverse operating systems.

```
       [ Gitea GitOps Repo (YAML Declarations) ]
                          │
                          ▼
             [ ArgoCD Deployment Engine ]
                          │
        ══════════ TAILSCALE MESH NETWORK ══════════
         /                │                    \
        ▼                 ▼                     ▼
 [ Proxmox VM ]     [ WSL2 Kernel ]     [ Windows Host ]
  (Ubuntu OS)       (Windows 11 PC)     (Native Windows)
        │                 │                     │
        ▼                 ▼                     ▼
  Linux Container   Linux Container     Windows Container
```

At the machine layer, top-level CLI tools hand off execution tasks to background daemons that coordinate directly with host kernels.

---

## 2. The Kernel Split — Linux Containers vs. Windows Containers

The absolute golden rule of containerization: **containers do not possess their own operating system kernel.** They run as isolated processes directly on the host machine's kernel.

### Linux Execution Layer

When a container image like `nginx:latest` or `gitea/gitea` runs on a Proxmox Ubuntu VM, the application inside talks directly to the living host Linux kernel. It leverages Linux **namespaces** (network and filesystem isolation) and **control groups** (cgroups, for resource limits).

### Windows Execution Layer

When a native Windows container runs on a Windows machine, the application binary makes system calls directly to the host Windows kernel (`ntoskrnl.exe`). It reads the Windows Registry, interacts with Windows Object Namespaces, and executes native Windows subsystem processes (`cmd.exe`, `powershell.exe`).

Because of this fundamental binary difference:

- You **cannot** run a standard Linux container image natively on a Windows kernel.
- You **cannot** run a native Windows container image on a Linux Proxmox host.

---

## 3. Deep Dive: Microsoft Virtualization & Isolation Modes

Because the open-source world compiles its applications almost exclusively for Linux, Microsoft engineered advanced translation systems inside Windows 11 and Windows Server to bridge the platform gap.

### The WSL2 Architecture (The Hybrid Linux Worker)

WSL2 does not emulate Linux commands. Instead, it runs a genuine, lightweight Linux kernel compiled by Microsoft inside a hidden utility hypervisor.

- **How it handles Kubernetes:** When you install the `k3s-agent` inside an Ubuntu WSL2 terminal, it binds to the WSL2 Linux kernel.
- To your Master nodes over Tailscale, **the WSL2 instance looks like a standard, native Linux worker node.** This allows a high-powered Windows workstation to run memory-intensive Linux pods natively.

### Native Windows Container Isolation Modes

If you run native Windows containers directly on the Windows 11 host via PowerShell, Microsoft utilizes two isolation modes depending on your OS version.

#### Mode 1: Process Isolation (Shared Kernel)

Mirrors traditional Linux containers. Multiple Windows containers run side-by-side, sharing the physical host Windows kernel.

> ⚠️ **Limitation:** Version matching must be exact. You cannot run a Windows Server 2022 container on a Windows 11 desktop host using Process Isolation due to kernel structure differences.

#### Mode 2: Hyper-V Isolation (Walled Kernel)

To fix version mismatching, Windows wraps the container in a highly optimized, microscopic **Hyper-V Virtual Machine** that boots in a fraction of a second. The container runs inside this micro-VM, which provides an isolated copy of the matching Windows kernel version it requires.

---

## 4. Geographic Boundaries & Latency: The Remote Worker Rule

Kubernetes allows you to register geographically remote nodes (e.g., a secondary office or remote site), but network latency alters your allowed deployment strategies.

```
[ Local Lab (Core Datacenter) ]                    [ Remote Location (Edge Node) ]
Master 1 (etcd) ── Worker 1                         Worker 5 (Remote Worker)
      │                                                   │
      └─── [Tailscale WireGuard Mesh] ────────────────────┘
```

### The Control Plane Consistency Engine (etcd)

The "brain" of a Kubernetes cluster relies on a highly sensitive distributed database called **etcd** to maintain cluster consensus.

> ⚠️ **The Latency Trap:** etcd requires ultra-low latency to validate transactions among cluster Masters. If round-trip communication exceeds **10ms–15ms**, etcd nodes drop consensus, causing the cluster to freeze or enter a protective read-only state.

> 🎯 **Design Rule:** Keep all Master nodes within your local high-speed switching fabric (`<1ms`). Geographically distant machines over public fiber connections (`30ms–50ms`) must be configured **strictly as Worker Nodes**.

### Storage Strategies for Remote Workers

High-speed synchronous replication tools like **Longhorn** will lock up and crash containers if forced to replicate block storage over a WAN connection.

- Remote worker edge environments must rely on **Local-Path Storage** (writing data directly to the node's local disk).
- Filesystem synchronization across long-distance boundaries must be handled asynchronously using tools like **Syncthing**.

---

## 5. Tailscale Mesh Network Topology

A major hurdle when joining Proxmox VMs, WSL2 subsystems, and remote physical environments is routing traffic through different firewalls and Carrier-Grade NAT (CGNAT) configurations without opening dangerous public ports.

```
[ Texas Master Node ]          [ WSL2 Local Node ]          [ Remote Edge Node ]
  Tailscale IP:                  Tailscale IP:                Tailscale IP:
  100.64.10.5                    100.64.10.12                 100.72.45.9
       │                              │                             │
       └──────────── [ WIREGUARD MESH ENVELOPE ] ──────────────────┘
```

By initializing the **Tailscale daemon on the base OS** of every participating node, each environment is assigned a permanent, encrypted CGNAT IP address within the `100.x.y.z` block.

When configuring **k3s**, networking flags are explicitly overridden to force cluster traffic over the encrypted Tailscale interfaces, treating all multi-platform and remote endpoints as though they are connected to a single flat internal switch.

---

## 6. Node Governance — Labels, Selectors, and Affinity

To prevent a Linux container from accidentally scheduling onto a native Windows node (causing an immediate crash), you must enforce declarative scheduling boundaries using **Labels** and **Node Selectors** in your YAML files.

When a node joins the cluster, Kubernetes automatically applies standard identification markers:

```yaml
# Examples of automatic and custom hardware tagging:
kubernetes.io/os: linux
kubernetes.io/os: windows
kubernetes.io/arch: amd64
topology.kubernetes.io/zone: remote-edge-site
```

### Enforcing Node Selection in Deployment Manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-linux-service
  namespace: infrastructure
spec:
  replicas: 2
  selector:
    matchLabels:
      app: core-service
  template:
    metadata:
      labels:
        app: core-service
    spec:
      nodeSelector:
        kubernetes.io/os: linux  # Forces cluster controller to ignore Windows worker nodes
      containers:
        - name: service-engine
          image: gitea/gitea:latest
          ports:
            - containerPort: 3000
```

---

## 7. The Windows Command Center — PowerShell Storage & Execution

### Enabling the Native Windows Virtualization Stack

To unlock the bare-metal Hyper-V hypervisor built into Windows 11 Pro or Windows Server, run from an **Administrative PowerShell Prompt**:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -All
```

> ⚠️ Requires a system restart to initialize the underlying Type-1 hypervisor layers.

### Deploying Hyper-V Virtual Machines via Command Line

```powershell
# 1. Define paths and allocate system memory and disk sizes
New-VM -Name "Lab-Worker-VM" -MemoryStartupBytes 8GB -Generation 2 -NewVHDPath "C:\HyperV-Disks\Worker.vhdx" -NewVHDSizeBytes 100GB

# 2. Append an installation ISO file
Add-VMDvdDrive -VMName "Lab-Worker-VM" -Path "C:\ISOs\ubuntu-live-server.iso"

# 3. Mount and kick off execution
Start-VM -Name "Lab-Worker-VM"

# 4. Monitor state
Get-VM | Select-Object Name, State, CpuUsage, MemoryAssigned
```

### Directing the Local Docker Desktop Daemon

```powershell
# Toggle Docker Daemon to native Windows Container mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Execute an isolated, native Windows command line sandbox instance
docker run -it mcr.microsoft.com/windows/nanoserver:ltsc2022 cmd.exe
```

> Inside this prompt, executing `dir` or `ipconfig` returns the interior namespace of a Windows container execution loop. Type `exit` to return to your Windows 11 host terminal.

---

## 8. Key Files, Paths, and Endpoints

### Windows Host Pathing

| Path | Purpose |
|------|---------|
| `C:\Program Files\Docker\Docker\DockerCli.exe` | Docker Desktop daemon control executable |
| `C:\ProgramData\Docker\config\daemon.json` | Native Windows Docker configuration file |
| `C:\Users\<User>\.wslconfig` | Global settings configuration file for WSL2 allocation |
| `\\wsl$\Ubuntu\` | Direct network path access to WSL2 Linux filesystem from Windows |

### Linux / k3s Cluster Pathing

| Path | Purpose |
|------|---------|
| `/etc/rancher/k3s/k3s.yaml` | Core cluster connection and credential configurations |
| `/var/lib/rancher/k3s/agent/` | k3s agent engine execution directories |
| `/var/lib/rancher/k3s/storage/` | Default local-path directory allocation for container data storage |

---

## 9. Worked Examples

### Example 1: Creating a Custom Resource-Limiting WSL2 Environment

By default, WSL2 can consume up to 80% of total system RAM. When running a heavy compute node like the Beelink (64GB DDR5), constrain its limits manually to keep the host OS performant.

```powershell
# Open or create the global configuration file
notepad "$env:USERPROFILE\.wslconfig"
```

Append the following resource constraints:

```ini
[wsl2]
memory=48GB           # Limit WSL2 to 48GB, leaving 16GB for Windows 11 Host
processors=12         # Constrain the virtual Linux kernel to 12 threads
localhostForwarding=true
```

Force-restart the subsystem to apply changes:

```powershell
# Shutdown all active WSL instances
wsl --shutdown

# Confirm state is cleanly stopped
wsl --list --verbose
```

---

### Example 2: Initializing a k3s Worker Agent Over a Tailscale Interface

To safely join the Beelink node into a remote Proxmox Master cluster, pass the Tailscale device IP flags during registration:

```bash
# Run inside your target Linux Worker Node / WSL2 Terminal:

# 1. Fetch your active Tailscale IP address
TAILSCALE_IP=$(tailscale ip -4)
echo "Registering node on IP: ${TAILSCALE_IP}"

# 2. Execute the k3s installer, pointing to your Master's Tailscale IP
curl -sfL https://get.k3s.io | K3S_URL="https://100.64.10.5:6443" \
  K3S_TOKEN="K10a76be04c538a7299042b938f30713bc4928374a8::server:secrettoken" \
  sh -s - agent \
  --node-ip="${TAILSCALE_IP}" \
  --flannel-iface="tailscale0" \
  --node-external-ip="${TAILSCALE_IP}"
```

---

### Example 3: Running Asynchronous Storage Replication for Remote Worker Nodes

Because network latency blocks real-time replication tools like Longhorn at remote locations, use local volumes paired with a background sync engine:

```yaml
# GitOps Manifest: /Deployments/syncthing-edge.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: syncthing-edge-worker
  namespace: storage-systems
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-sync
  template:
    metadata:
      labels:
        app: data-sync
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: remote-edge-site  # Forces deployment to remote node
      containers:
        - name: syncthing
          image: syncthing/syncthing:latest
          volumeMounts:
            - mountPath: /var/syncthing
              name: local-storage-pool
      volumes:
        - name: local-storage-pool
          hostPath:
            path: /var/lib/rancher/k3s/storage/edge-data-dir
            type: DirectoryOrCreate
```

---

## 10. Troubleshooting Multi-OS Scenarios

### Scenario 1: Pod stuck in `ImagePullBackOff` or `CrashLoopBackOff` on Windows Node

**Symptom:** An application schedules onto the native Windows node but immediately fails with a platform execution or image layout error.

**Root Cause:** A standard Linux container image was scheduled to a native Windows kernel worker node because a `nodeSelector` was omitted from the YAML manifest.

**Fix:** Delete the broken pod and add the OS target to your Gitea deployment:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux  # Or 'windows' depending on the application binary
```

---

### Scenario 2: Remote Worker Node constantly shows `NotReady` Status

**Symptom:** A node at a secondary location joins the cluster successfully but frequently drops offline or enters `NotReady` in the dashboard.

**Root Cause:** Cluster communication is attempting to pass traffic over local router IPs (`192.168.1.x`) instead of binding through the Tailscale interface.

**Fix:** Remote logs will show a dial timeout to the control plane. Log into the affected node and ensure `--flannel-iface=tailscale0` and `--node-ip` match your exact `100.x.y.z` network footprint.

---

## 11. Architectural Gotchas

1. **The Core Brain Constraints:** You cannot run HA k3s master nodes (etcd) across WAN connections. Keep your control plane inside a tight, sub-millisecond local network loop to guarantee data consistency.

2. **Container Engine Control Hijacks:** Running native Windows containers requires switching the active background engine daemon via PowerShell. If your Docker Compose layouts throw directory formatting errors, you likely forgot to execute `SwitchDaemon` back to Linux mode.

3. **Disk Allocation Overwrite Hazards:** WSL2 filesystems are stored inside a virtual expandable disk (`ext4.vhdx`). If containers generate large temporary data logs, this file can expand rapidly and consume your primary Windows host SSD until manually compacted via `diskpart`.

4. **Network Namespace Clashes:** Windows containers do not natively share a single host routing path like Linux processes. Each Windows pod spins up its own internal virtual network adapter endpoint, requiring proper allocation parameters inside your Flannel/Calico CNI configurations.

5. **Ingress Controllers Are Linux-Only — Pin Them Explicitly.** Traefik, Nginx Ingress, and every other mainstream ingress controller is compiled as a Linux binary. In a hybrid cluster with both Linux and Windows worker nodes, an ingress controller deployed without a `nodeSelector` may attempt to schedule on a Windows node, fail immediately, and leave your cluster with no ingress path — often with a confusing `Pending` or `CrashLoopBackOff` state that does not obviously point to an OS mismatch.

   Always enforce Linux placement on ingress controllers explicitly, even though it feels redundant:

   ```yaml
   # Traefik / Nginx Ingress deployment spec — always include this
   spec:
     template:
       spec:
         nodeSelector:
           kubernetes.io/os: linux
   ```

   If you are deploying via Helm, pass the nodeSelector as a value override:

   ```bash
   # Traefik via Helm — force Linux nodes
   helm install traefik traefik/traefik \
     --set nodeSelector."kubernetes\\.io/os"=linux \
     --namespace kube-system

   # Nginx Ingress via Helm
   helm install ingress-nginx ingress-nginx/ingress-nginx \
     --set controller.nodeSelector."kubernetes\\.io/os"=linux \
     --namespace ingress-nginx
   ```

   > ⚠️ **The secondary risk:** Even if the ingress controller schedules correctly on a Linux node, it must still proxy traffic to Windows pod IPs across the Flannel VXLAN overlay. This works — but only if UDP 4789 is open on the Windows node's Tailscale adapter firewall (see `windows-linux-containers-vms-complete.md` Section 14). An ingress controller that can reach Linux pods but silently times out on Windows pods is almost always the UDP 4789 firewall rule missing on the Windows node.

---

## 12. Homelab Practice Tasks

### Task 1: Initialize Hyper-V Layer Control

```powershell
# a. Verify Hyper-V capabilities
Get-WindowsOptionalFeature -Online | Where-Object {$_.FeatureName -like "*Hyper-V*"}

# b. Practice creating a Generation 2 VM via PowerShell (no GUI)
```

### Task 2: Configure a Constrained Subsystem Boundary

```powershell
# a. Build a custom .wslconfig in your user directory
# b. Clamp memory limits to exactly 50% of total physical memory
# c. Recycle the background layers
wsl --shutdown

# d. Verify the OS respects boundary limits
wsl -e free -h
```

### Task 3: Map Declarative Edge Routing Over Tailscale

```bash
# a. Examine cluster node status
kubectl get nodes -o wide

# b. Verify INTERNAL-IP column reflects 100.x.y.z Tailscale mesh IPs
#    (not standard 192.168.x.x router addresses)
```

### Task 4: Practice Container Mode Toggling

```powershell
# a. Launch PowerShell as Administrator, switch Docker daemon to Windows mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# b. Run a native nanoserver sandbox
docker run -it mcr.microsoft.com/windows/nanoserver:ltsc2022 cmd.exe

# c. Switch back to Linux mode and verify Docker Compose stacks restore connectivity
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
```

---

## Quick Reference

### PowerShell Commands

| Command | Purpose |
|---------|---------|
| `Get-VM` | Lists all local Hyper-V virtual machines |
| `Start-VM -Name "XYZ"` | Boots a specific virtual machine |
| `Stop-VM -Name "XYZ" -Force` | Hard kills an active virtual machine instance |
| `wsl --shutdown` | Instantly kills all active background WSL2 Linux kernels |
| `wsl --list --verbose` | Identifies active state and versions of WSL distributions |
| `& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon` | Switches between Windows and Linux container execution loops |

### Core Infrastructure Components

| Component | Operational Focus |
|-----------|------------------|
| **Control Plane (Master Nodes)** | Must run on **Linux**. Houses the etcd configuration matrix. |
| **WSL2 Worker Node** | Hybrid layer. Runs genuine **Linux workloads** inside a lightweight hypervisor on Windows. |
| **Native Windows Worker Node** | Runs **Windows native containers** directly against the host Windows kernel. |
| **Tailscale Mesh Interconnect** | Handles site-to-site VPN routing, transforming all endpoints into a flat `100.x.y.z` map. |

---

*File: `Kubernetes-hybrid-architecture.md`*  
*See also: `windows-linux-containers-vms-complete.md` | `windows-dockerfile-patterns.md`*
