# Homelab Docs

Technical reference documentation built from real homelab experience running a hybrid Proxmox/Kubernetes/Windows infrastructure.

## Contents

### Kubernetes
- [Kubernetes Hybrid Architecture](docs/kubernetes/Kubernetes-hybrid-architecture.md) — Hybrid OS cluster design, etcd boundaries, Tailscale mesh, WSL2 workers, node governance

### Containers
- [Windows & Linux Containers and VMs Complete Reference](docs/containers/windows-linux-containers-vms-complete.md) — Deep OS-level reference covering both platforms, Registry, HNS, PowerShell, cross-platform connectivity
- [Windows Dockerfile Patterns](docs/containers/windows-dockerfile-patterns.md) — Image engineering, service patterns, layer optimization, multi-stage builds, Registry manipulation

## Stack

- Proxmox VE — 6 node bare metal cluster
- k3s — Kubernetes across hybrid Linux and Windows nodes
- Tailscale — WireGuard mesh across all nodes
- Windows/Linux hybrid workloads
- Oracle Cloud edge node

## About

Built by a homelabber learning from the ground up. Everything documented here comes from actually building and breaking real systems.
