# Physical Isolation (Out-of-Band Management)

## Core Concept

Physical isolation is the **simplest and most absolute** form of network separation: completely different physical hardware. In an NVIDIA AI Factory, **OOB (Out-of-Band) management** traffic is the one category that does **not** ride on the Common Networking infrastructure. It uses:

- Separate NICs (dedicated BMC/IPMI port on each server)
- Separate switches (typically basic 1GbE management switches)
- Separate cabling

This establishes the baseline that the most critical, lowest-level management (hardware control) is **never** exposed to the Common Networking fabric—regardless of how well it's segmented logically.

## Traffic Separation

| OOB Network (Physical Isolation) | Common Networking (Logical Isolation) |
|----------------------------------|---------------------------------------|
| IPMI/Redfish commands | Kubernetes pod traffic |
| Serial-over-LAN console | Storage I/O (NFS, RoCEv2) |
| Firmware updates | Inference API requests |
| Power control | In-band SSH/management |
| Hardware health monitoring | Tenant workload traffic |

## Why Physical Separation for OOB?

| Reason | Explanation |
|--------|-------------|
| **Security** | BMC access = full hardware control (power, console, BIOS). A compromised BMC means game over. Keep it unreachable from tenant networks. |
| **Availability** | If the Common Networking fabric has issues, you still need console access to troubleshoot. OOB stays up independently. |
| **Simplicity** | BMC interfaces are typically 1GbE with basic requirements. No need for VXLAN, QoS, or sophisticated features. |

## Topology Options and Trade-offs

OOB traffic doesn't traverse the Spectrum-X spine-leaf fabric (the high-bandwidth data path to the DPUs), but it must reach the outside world somehow. Two approaches:

### Option A: OOB VLANs on Spine Switches

```
BMC Ports ──► OOB VLANs on Spectrum-X Spines ──► Border Router
DPU Ports ──► Data VLANs on Spectrum-X Spines ──┘
```

| Pros | Cons |
|------|------|
| Fewer switches to buy/manage | **Coupled failure domain** - spine issue = lose OOB when you need it most |
| Simplified cabling | **Security risk** - shared L2 between BMC and tenant data |
| Single management plane | **Blast radius** - bad config push affects data AND management |
| Less rack space | Wasting 400G ports on 1GbE BMC traffic |
| | Can't maintain spines without affecting OOB access |

### Option B: Separate OOB, IP Connectivity Upstream (Recommended)

```
BMC Ports ──► Dedicated 1GbE Switches ──► Border Router (VLAN/VRF isolated)
DPU Ports ──► Spectrum-X Spine-Leaf ────┘
```

| Pros | Cons |
|------|------|
| **Independent failure domains** - OOB up even if Spectrum-X is down | More hardware to purchase |
| **Security** - no shared L2 with data plane | More cabling |
| **Operational independence** - upgrade fabric without losing console | Two switching infrastructures to manage |
| Simple 1GbE switches, no VXLAN/EVPN complexity | |
| Blast radius containment | |

### Recommendation

**Separate OOB is worth the extra switches.** You need OOB access most when the data fabric is broken. If OOB rides on the same infrastructure, you lose management access precisely when you need it. The upstream border router becomes a shared point, but it's typically hardened, stable, L3 infrastructure outside the blast radius of fabric changes.

## Terminology

### Common Networking vs. North-South Network

These terms describe the **same physical fabric** from different perspectives:

| Term | Perspective | Meaning |
|------|-------------|---------|
| **North-South Network** | Traffic direction | Traffic flowing vertically through the spine-leaf topology—into/out of the cluster (inference requests), up to storage, down to hosts. Contrasts with **East-West** (GPU-to-GPU lateral traffic on the dedicated compute fabric). |
| **Common Networking** | Architecture/purpose | A shared infrastructure serving multiple traffic types (storage, K8s, inference, in-band mgmt) with logical isolation. Emphasizes that one fabric handles many roles. |

```
                         NORTH (External/Upstream)
                                  ▲
                                  │
                         ┌────────┴────────┐
                         │  Spine Switches │
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
       ┌──────┴──────┐     ┌──────┴──────┐     ┌──────┴──────┐
       │    Leaf     │     │    Leaf     │     │    Leaf     │
       └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
              │                   │                   │
              ▼                   ▼                   ▼
         SOUTH (Hosts/DPUs)

        ════════════════════════════════════════════════
        This entire fabric is both:
        • "North-South Network" (traffic direction)
        • "Common Networking" (shared purpose)
        ════════════════════════════════════════════════
```

Use **North-South** when contrasting with the East-West GPU fabric. Use **Common Networking** when emphasizing multi-tenancy and logical isolation.

### Common Networking vs. Converged Ethernet

| Term | Meaning |
|------|---------|
| **Common Networking** | A shared physical infrastructure carrying multiple traffic types (storage, K8s, inference, in-band management) with logical isolation. The Spectrum-X spine-leaf fabric is "common" in this sense—one fabric serving many purposes. |
| **Converged Ethernet** | Ethernet enhanced with **DCB (Data Center Bridging)** standards—PFC (Priority Flow Control), ETS (Enhanced Transmission Selection), DCBx—enabling lossless transport for protocols like RoCEv2. A capability of the Ethernet fabric, not an architecture. |

## Key Takeaways

1. **Physical isolation** keeps BMC/OOB traffic on entirely separate infrastructure from the Common Networking fabric
2. **Separate OOB switches** are recommended over VLAN carve-outs on spine switches for failure domain independence
3. **Common Networking** = shared fabric, logically isolated (same as North-South Network)
4. **Converged Ethernet** = DCB-enhanced Ethernet for lossless transport (a capability, not an architecture)
5. Everything else we'll discuss (VLANs, VXLAN, VRF, etc.) applies to the **Common Networking** fabric where logical isolation is used
