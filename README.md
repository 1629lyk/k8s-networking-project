# Kubernetes Networking Deep-Dive

> **Goal:** Convert Kubernetes comfort into network engineer value.  
> **Stack:** Ubuntu 22.04 WSL2 · kind · Calico CNI · etcd Raft  
> **Outcome:** Hands-on mastery of pod networking, NetworkPolicy, service reachability, packet capture, and distributed system failure modes.

---

## Why This Project

Many engineering teams expect platform and network engineers to understand Kubernetes traffic paths and policy controls at a deep level — not just "deploy and hope." This project builds that understanding from scratch, layer by layer, with real packet captures and real failure documentation.

---

## Project Structure

```
k8s-networking-project/
├── README.md
├── phase1-environment-setup.md
├── phase2-calico-cni.md
├── phase3-networkpolicy.md
├── phase4-services-dns.md
├── phase5-packet-capture.md
├── phase6-observability.md
├── phase7-raft-failures.md
└── issues.md
```

---

## Phases Overview

| Phase | Topic | Key Skills |
|-------|-------|------------|
| [Phase 1](phase1-environment-setup.md) | Environment Setup | Docker, kind, kubectl, helm, WSL2 quirks |
| [Phase 2](phase2-calico-cni.md) | Calico CNI | Pod IPs, cross-node routing, veth interfaces |
| [Phase 3](phase3-networkpolicy.md) | NetworkPolicy | Zero-trust, 3-tier segmentation, silent drops |
| [Phase 4](phase4-services-dns.md) | Services & DNS | ClusterIP, kube-proxy DNAT, CoreDNS |
| [Phase 5](phase5-packet-capture.md) | Packet Capture | tcpdump, conntrack, nsenter |
| [Phase 6](phase6-observability.md) | Observability | Goldpinger, cross-node health |
| [Phase 7](phase7-raft-failures.md) | Raft & Failures | etcd cluster, 3 failure modes documented |
| [Issues](issues.md) | Known Issues | 10 documented WSL2/kind/etcd issues with fixes |

---

## Prerequisites

- Windows 10/11 with WSL2 enabled
- Ubuntu 22.04 installed in WSL2
- At least 8GB RAM (16GB recommended)
- 20GB free disk space
- Internet connection for image pulls

---

## Quick Start

```bash
# Clone this repo
git clone <your-repo-url>

# Start from Phase 1 and follow each phase in order
# Each phase builds on the previous one
```

> ⚠️ **WSL2 Note:** `kubectl` direct access is broken on WSL2 due to Docker bridge networking.  
> This runbook uses `alias k="docker exec k8s-net-lab-control-plane kubectl"` throughout.  
> Set this up in Phase 1 and use `k` for all kubectl commands.

---

## Key Concepts Covered

### Networking
- How Calico assigns pod IPs and programs routes between nodes
- How kube-proxy translates ClusterIP → real pod IP via iptables DNAT
- How CoreDNS auto-wires every pod with cluster DNS
- How NetworkPolicy creates kernel-level packet filtering

### Observability
- Live packet capture with `tcpdump` via `nsenter` (no tools on node needed)
- Connection state inspection with `conntrack`
- DNS query tracing showing dual A+AAAA lookups
- Cross-node health monitoring with Goldpinger

### Distributed Systems
- Raft leader election triggered by node failure
- DNS exclusion of NotReady pods (correct behavior, not a bug)
- Bootstrap conflict from missing PersistentVolumeClaims
- Zero data loss during leadership transitions

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `kind` | Kubernetes IN Docker — multi-node local cluster |
| `kubectl` | Kubernetes API client |
| `helm` | Kubernetes package manager |
| `calico` | CNI plugin — pod networking and NetworkPolicy |
| `tcpdump` | Packet capture |
| `conntrack` | Connection tracking table inspection |
| `nsenter` | Enter network namespaces without node tools |
| `etcdctl` | etcd cluster management and health checks |
| `goldpinger` | Cross-node pod connectivity health dashboard |

---

## Failure Modes Documented

1. **Leader Node Loss** — Raft election, zero data loss, <4ms recovery
2. **NotReady Pod DNS Exclusion** — Pod Running but excluded from DNS correctly
3. **Bootstrap Conflict** — StatefulSet without PVCs causes crash loop on restart

---

## Environment Details

Tested on:
- Ubuntu 22.04.5 LTS (WSL2)
- Kernel: 5.15.153.1-microsoft-standard-WSL2
- kind: v0.31.0
- kubectl: v1.35.3
- Calico: v3.27.0
- etcd: v3.5.0
