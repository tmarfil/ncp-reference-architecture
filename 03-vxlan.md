# VXLAN (Virtual Extensible LAN)

## Core Concept

VXLAN solves two fundamental VLAN limitations:

1. **4094 segment limit** — not enough for large multi-tenant environments
2. **L2 boundary** — VLANs don't cross routers

VXLAN encapsulates Layer 2 frames inside Layer 3 UDP packets, providing:
- **24-bit VNI** (VXLAN Network Identifier) = 16 million segments
- **L2 over L3** = overlay networks that span routed boundaries

## Encapsulation Format

```
   Original L2 Frame:
   ┌──────────┬──────────┬───────────┬─────────┐
   │ Dest MAC │ Src MAC  │ EtherType │ Payload │
   └──────────┴──────────┴───────────┴─────────┘
                         │
                         ▼
   VXLAN Encapsulated:
   ┌──────────┬─────────┬─────────┬──────────┬─────────────────────┐
   │ Outer L2 │Outer IP │Outer UDP│  VXLAN   │  Original L2 Frame  │
   │  Header  │ Header  │ :4789   │  Header  │     (unchanged)     │
   └──────────┴─────────┴─────────┴────┬─────┴─────────────────────┘
                                       │
                              ┌────────┴────────┐
                              │ 24-bit VNI      │
                              │ (16M segments)  │
                              └─────────────────┘
```

## Key Terminology

| Term | Meaning |
|------|---------|
| **VNI** | VXLAN Network Identifier — 24-bit segment ID (0–16,777,215) |
| **VTEP** | VXLAN Tunnel Endpoint — device that encapsulates/decapsulates |
| **Underlay** | Physical IP network carrying encapsulated VXLAN traffic |
| **Overlay** | Virtual L2 network created by VXLAN (what tenants see) |

## Overlay vs. Underlay

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           OVERLAY                                       │
│  What tenants see: L2 segments spanning the entire fabric              │
│  • Tenant A: VNI 10001 — appears as one broadcast domain               │
│  • Tenant B: VNI 10002 — isolated, different broadcast domain          │
│  • Pods/VMs can ARP, broadcast as if L2 adjacent                       │
├─────────────────────────────────────────────────────────────────────────┤
│                           UNDERLAY                                      │
│  Physical reality: Routed IP network (spine-leaf)                      │
│  • VTEP-to-VTEP communication via IP routing                           │
│  • ECMP across multiple spine paths                                     │
│  • No spanning tree, full bandwidth utilization                        │
└─────────────────────────────────────────────────────────────────────────┘
```

The underlay is typically **routed L3** (not a VLAN). VTEPs have IP addresses and VXLAN packets are routed between them. VLANs may appear at the leaf-to-host edge, but the spine-leaf core is pure IP routing.

## VTEP Location Patterns

| Pattern | VTEP Location | Use Case |
|---------|---------------|----------|
| **Network-centric** | Leaf switch | Traditional enterprise, hosts see VLANs |
| **Host-centric** | Hypervisor / K8s node | VMware NSX, KVM+OVS, overlay CNIs |
| **DPU-accelerated** | BlueField DPU | AI Factory, hardware offload at host edge |

The industry trend is **host-centric with hardware offload**: VTEP pushed to the edge (hypervisor, node, or DPU), physical switches just route IP.

## VNI-to-VLAN Mapping at the Edge

At host-facing ports, a VNI maps to a local VLAN:

```
   Host                    Leaf Switch / DPU
┌─────────┐              ┌───────────────────────────────┐
│         │              │                               │
│ VLAN 100├──────────────┤ Access VLAN 100               │
│ (local) │  untagged    │       ↓                       │
│         │              │ Maps VLAN 100 → VNI 10001     │
└─────────┘              │       ↓                       │
                         │ VTEP encapsulates for fabric  │
                         └───────────────────────────────┘
```

Different leaves can use different local VLAN IDs for the same VNI—the VNI is the canonical identifier across the fabric.

## Tenant Isolation via VNI

VNI is the isolation boundary. Tenants on the same physical host are isolated:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 SAME PHYSICAL HOST, DIFFERENT TENANTS                   │
│                                                                         │
│   Tenant A Pods          Tenant B Pods                                 │
│   10.0.1.0/24            10.0.1.0/24      ◄── Same IP range, isolated  │
│        │                      │                                         │
│   VNI 10001              VNI 20001                                      │
│        │                      │                                         │
│   ┌────┴──────────────────────┴────┐                                   │
│   │       BlueField DPU (VTEP)     │                                   │
│   │  Encapsulates with different   │                                   │
│   │  VNIs - complete isolation     │                                   │
│   └────────────────────────────────┘                                   │
│                                                                         │
│   Tenant A cannot see Tenant B traffic, even on shared hardware        │
└─────────────────────────────────────────────────────────────────────────┘
```

## VMware/KVM vs. K8s: Same Pattern

| Environment | VTEP Implementation | Control Plane |
|-------------|---------------------|---------------|
| **VMware NSX** | ESXi kernel module | NSX Manager |
| **KVM + OpenStack** | OVS on host | Neutron |
| **K8s (overlay CNI)** | Flannel/Calico/Cilium | CNI daemon |
| **K8s + BlueField** | DPU hardware (ASAP²) | EVPN or DOCA |

The fundamental pattern is identical: VTEP at the host edge, physical fabric routes IP.

## BlueField DPU Capabilities

| Feature | Implementation |
|---------|----------------|
| **VXLAN VTEP** | Hardware-accelerated via ASAP² / OVS-DOCA (line rate) |
| **BGP EVPN** | Software on Arm cores (FRRouting) — optional |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      BLUEFIELD DPU ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│   Arm Cores (Software)                                                  │
│   • Linux OS, FRRouting (BGP EVPN), DOCA Services                      │
│   • Control plane: pushes MAC/IP→VNI→VTEP mappings to hardware         │
├─────────────────────────────────────────────────────────────────────────┤
│   ConnectX Silicon (Hardware)                                           │
│   • ASAP² / eSwitch                                                     │
│   • VXLAN encap/decap at line rate                                     │
│   • No CPU involvement for data plane                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Control Plane vs. Data Plane: Standard Linux vs. DOCA

| Layer | Component | Standard Linux | DOCA-Accelerated |
|-------|-----------|----------------|------------------|
| **Control Plane** | BGP/EVPN | FRRouting (`apt install frr`) | N/A - not needed |
| **Data Plane** | Switching | OVS-Kernel | **OVS-DOCA** (ASAP²) |
| **Data Plane** | VXLAN | Kernel VXLAN | **OVS-DOCA VXLAN** |
| **Data Plane** | Firewall | iptables/nftables | **DOCA Firewall** |

**Key Insight**: Control plane (BGP, routing) is standard Linux software—portable to any ARM system. Data plane (packet forwarding, encapsulation) benefits from DOCA hardware acceleration.

See [Appendix: DOCA Reference](appendix-doca-reference.md) for complete component mapping and Kubernetes CRDs.

## DPU Management Models

| Environment | DPU Management | Approach |
|-------------|----------------|----------|
| **K8s (inference)** | NVIDIA DPF Operator + CRDs | K8s-native, GitOps |
| **Slurm (training)** | Ansible + SSH, or K8s on DPU only | Direct Linux config |
| **Hybrid** | DPU runs isolated K3s | Separate infra plane |

For Slurm-based training clusters (no K8s on hosts), DPU management options:
- Direct SSH + Ansible (most common)
- K8s running isolated on DPU Arm cores
- DOCA containers without K8s orchestration

## Training vs. Inference Orchestration

| Workload | Orchestrator | Why | DPU Role |
|----------|--------------|-----|----------|
| **Training** | Slurm | Gang scheduling, minimal overhead, MPI | Storage connectivity, basic networking |
| **Inference** | Kubernetes | Auto-scaling, multi-tenancy, rolling updates | Multi-tenant isolation, VXLAN VTEP, traffic enforcement |

K8s dominates inference due to auto-scaling, multi-tenancy, rolling updates, and API gateway patterns. VXLAN/VNI isolation becomes critical for multi-tenant inference.

## The Control Plane Problem

VXLAN defines data plane encapsulation but not how VTEPs learn:
- Which remote VTEPs exist?
- Which MAC addresses are behind which VTEP?
- Which VNIs are active where?

**Options:**
1. **Flood and learn** — doesn't scale
2. **Static configuration** — operational nightmare
3. **BGP EVPN** — standards-based, distributed control plane (the right answer)

## Cumulus Linux Example

```bash
# Configure VTEP source address (loopback)
nv set nve vxlan source address 10.0.0.1

# Map VLAN 100 to VNI 10001
nv set bridge domain br_default vlan 100 vni 10001

# Access port for host
nv set interface swp1 bridge domain br_default access 100

# Apply configuration
nv config apply

# The control plane (BGP EVPN) handles VTEP discovery - covered in next topic
```

## Key Takeaways

1. **VXLAN** encapsulates L2 over L3, providing 16M segments and fabric-wide L2 domains
2. **VNI** is the isolation boundary for multi-tenancy—same IP ranges can be reused across tenants
3. **Underlay is routed IP**, not VLANs (VLANs appear only at the edge)
4. **VTEP at host edge** (DPU) is the modern pattern—physical switches just route
5. **BlueField DPUs** provide hardware VXLAN offload (ASAP²); BGP EVPN runs as standard Linux software (FRR)
6. **Control plane** (BGP/EVPN) = standard Linux; **Data plane** (encap/decap) = DOCA-accelerated
7. **Slurm for training, K8s for inference**—DPU management model differs accordingly
8. **BGP EVPN** solves the control plane problem (next topic)
