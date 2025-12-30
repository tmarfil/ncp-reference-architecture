# BGP EVPN (Border Gateway Protocol - Ethernet VPN)

## Core Concept

VXLAN defines **data plane encapsulation** but leaves a critical question unanswered:

> **How do VTEPs learn about each other and the MAC/IP addresses behind them?**

**BGP EVPN** is the answer: a standards-based control plane that distributes MAC and IP reachability information via BGP.

| Component | What It Is |
|-----------|------------|
| **BGP** | Border Gateway Protocol — distributes reachability information between peers |
| **EVPN** | Ethernet VPN — an address family (AFI/SAFI) within BGP that carries L2 (MAC) and L3 (IP) information for overlay networks |

## Two Separate BGP Domains

In DPU-centric architectures, there are **two distinct BGP domains**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                TWO BGP DOMAINS (Underlay vs Overlay)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   UNDERLAY BGP (eBGP)                   OVERLAY BGP EVPN (iBGP)        │
│   ───────────────────                   ────────────────────────        │
│                                                                         │
│   Purpose: VTEP IP reachability         Purpose: MAC/IP→VTEP mapping   │
│                                                                         │
│   Peers: Leaf ↔ Spine                   Peers: DPU ↔ RR (Spines)       │
│                                                                         │
│   AS: Different per leaf (eBGP)         AS: Same across all DPUs       │
│                                             (iBGP)                      │
│                                                                         │
│   Routes: /32 loopbacks                 Routes: Type-2 (MAC+IP)        │
│           Link subnets                          Type-3 (VTEP discovery)│
│                                                 Type-5 (IP Prefix)     │
│                                                                         │
│   Learned by: Leaves, Spines, DPUs      Learned by: DPUs               │
│                                                                         │
│   Used for: Outer IP routing            Used for: Inner frame lookup   │
│             (VXLAN encap'd packets)             (which VTEP owns MAC?) │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Underlay BGP

Establishes IP connectivity between VTEPs via eBGP on point-to-point routed links:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BGP UNDERLAY                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    ┌──────────────────────┐                            │
│                    │       SPINES         │                            │
│                    │      AS 65000        │  ← Shared AS               │
│                    └──────────┬───────────┘                            │
│                               │                                         │
│                          eBGP │ point-to-point /31 links               │
│                               │ (routed, not L2)                       │
│                               │                                         │
│         ┌─────────────────────┼─────────────────────┐                  │
│         │                     │                     │                   │
│    ┌────┴────┐           ┌────┴────┐           ┌────┴────┐             │
│    │ Leaf 1  │           │ Leaf 2  │           │ Leaf 3  │             │
│    │ AS 65001│           │ AS 65002│           │ AS 65003│             │
│    │Lo:10.0.0.1          │Lo:10.0.0.2          │Lo:10.0.0.3            │
│    └────┬────┘           └────┬────┘           └────┬────┘             │
│         │                     │                     │                   │
│    ┌────┴────┐           ┌────┴────┐           ┌────┴────┐             │
│    │   DPU   │           │   DPU   │           │   DPU   │             │
│    │Lo:10.0.1.1          │Lo:10.0.1.2          │Lo:10.0.1.3            │
│    │ (VTEP)  │           │ (VTEP)  │           │ (VTEP)  │             │
│    └─────────┘           └─────────┘           └─────────┘             │
│                                                                         │
│   PURPOSE: IP reachability between VTEP loopbacks                      │
│   ROUTES: Loopback /32s (VTEP addresses)                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Overlay BGP EVPN

Distributes MAC/IP → VTEP mappings via iBGP with route reflectors:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BGP EVPN OVERLAY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    ┌──────────────────────┐                            │
│                    │       SPINES         │                            │
│                    │   (Route Reflectors) │                            │
│                    │      iBGP EVPN       │                            │
│                    │      AS 65000        │                            │
│                    └──────────┬───────────┘                            │
│                               │                                         │
│                          iBGP │ EVPN sessions                          │
│                               │ (same AS across all EVPN speakers)     │
│                               │                                         │
│         ┌─────────────────────┼─────────────────────┐                  │
│         │                     │                     │                   │
│    ┌────┴────┐           ┌────┴────┐           ┌────┴────┐             │
│    │   DPU   │           │   DPU   │           │   DPU   │             │
│    │  VTEP   │           │  VTEP   │           │  VTEP   │             │
│    │ AS 65000│           │ AS 65000│           │ AS 65000│             │
│    │(iBGP)   │           │(iBGP)   │           │(iBGP)   │             │
│    └─────────┘           └─────────┘           └─────────┘             │
│                                                                         │
│   PURPOSE: Distribute MAC/IP → VTEP + VNI mappings                     │
│   ROUTES: Type-2 (MAC+IP), Type-3 (VTEP), Type-5 (prefix)             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## EVPN Route Types

| Type | Name | Purpose | Content |
|------|------|---------|---------|
| **Type-2** | MAC/IP Advertisement | "This MAC (and optionally IP) is behind this VTEP in this VNI" | RD, RT, MAC, IP, VNI, Next-hop VTEP |
| **Type-3** | Inclusive Multicast Ethernet Tag | "This VTEP participates in this VNI" (VTEP discovery) | RD, RT, VNI, Originating Router |
| **Type-5** | IP Prefix | "This IP prefix is reachable via this VTEP" (L3 routing) | RD, RT, IP Prefix, Gateway IP, VNI |

### Type-2 Route Example

```
Route Distinguisher: 10.0.1.1:100
Route Target: 65000:10001
MAC: aa:bb:cc:dd:ee:ff
IP: 10.100.1.10 (optional)
VNI: 10001
Next-hop (VTEP): 10.0.1.1

→ "MAC aa:bb:cc:dd:ee:ff with IP 10.100.1.10 in VNI 10001 is behind VTEP 10.0.1.1"
```

## Traffic Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TRAFFIC FLOW                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   VM-A (10.100.1.10, MAC: aa:aa)        VM-B (10.100.1.20, MAC: bb:bb) │
│   VNI: 10001                             VNI: 10001                     │
│        │                                      ▲                         │
│        │ 1. VM-A sends frame to VM-B          │                         │
│        │    dst MAC: bb:bb                    │                         │
│        ▼                                      │                         │
│   ┌─────────────────────────────────────┐     │                         │
│   │            DPU-A (VTEP: 10.0.1.1)   │     │                         │
│   │                                     │     │                         │
│   │  2. DPU looks up dst MAC bb:bb      │     │                         │
│   │     in EVPN-learned FIB:            │     │                         │
│   │                                     │     │                         │
│   │     MAC bb:bb, VNI 10001            │     │                         │
│   │       → VTEP 10.0.1.2               │     │                         │
│   │                                     │     │                         │
│   │  3. DPU encapsulates:               │     │                         │
│   │     Outer IP: 10.0.1.1 → 10.0.1.2   │     │                         │
│   │     UDP: 4789                        │     │                         │
│   │     VXLAN VNI: 10001                │     │                         │
│   │     Inner: Original frame unchanged │     │                         │
│   │                                     │     │                         │
│   │  4. DPU uses UNDERLAY routing to    │     │                         │
│   │     reach 10.0.1.2                  │     │                         │
│   └──────────────────┬──────────────────┘     │                         │
│                      │                        │                         │
│                      ▼                        │                         │
│          UNDERLAY: Leaf → Spine → Leaf        │                         │
│          (regular IP routing to 10.0.1.2)     │                         │
│                      │                        │                         │
│                      ▼                        │                         │
│   ┌─────────────────────────────────────┐    │                         │
│   │            DPU-B (VTEP: 10.0.1.2)   │    │                         │
│   │                                     │    │                         │
│   │  5. DPU-B decapsulates:             │    │                         │
│   │     - Strips outer headers          │    │                         │
│   │     - Validates VNI 10001           │    │                         │
│   │     - Delivers original frame       │────┘                         │
│   └─────────────────────────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Route Reflector Architecture

Without route reflectors, every DPU needs to peer with every other DPU (full mesh). Route reflectors simplify this:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ROUTE REFLECTOR TOPOLOGY                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   WITHOUT RR (Full Mesh)              WITH RR (Spines as RR)           │
│   ──────────────────────              ─────────────────────             │
│                                                                         │
│   D1 ◄──► D2                          D1      D2      D3      D4       │
│    ▲  ╲  ╱  ▲                          │       │       │       │        │
│    │   ╲╱   │                          │       │       │       │        │
│    │   ╱╲   │                          ▼       ▼       ▼       ▼        │
│    ▼  ╱  ╲  ▼                       ┌─────────────────────────────┐     │
│   D3 ◄──► D4                        │  Spine 1 (RR)  Spine 2 (RR) │     │
│                                     └─────────────────────────────┘     │
│   N*(N-1)/2 sessions                                                    │
│   100 DPUs = 4,950 sessions          100 DPUs = 100 sessions           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## EVPN Deployment Models (Ranked Best to Worst)

### Option 1: DPU-Centric + Spine RR (BEST)

```
┌─────────────────────────────────────────────────────────────────────────┐
│   Spines: eBGP underlay + iBGP EVPN RR (no VTEP)                       │
│   Leaves: eBGP underlay only (no EVPN, no VTEP)                        │
│   DPUs:   iBGP EVPN + VTEP (hardware accelerated)                      │
└─────────────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| DPU hardware VXLAN offload (line rate) | DPU must run BGP (FRR) |
| Host OS sees single NIC, no bonding | |
| Hardware ECMP on DPU | |
| Leaves are simple L3 routers | |
| Scales well | |

### Option 2: DPU-Centric + Dedicated RR Cluster

| Pros | Cons |
|------|------|
| RR completely out of forwarding path | Additional infrastructure |
| Can scale RR independently | More things to manage |
| Best for hyperscale (1000s of DPUs) | |

### Option 3: DPU-Centric + Leaf RR

| Pros | Cons |
|------|------|
| DPU peers with directly connected leaf | Leaves more complex |
| Low latency to RR | Leaf failure affects local DPUs |

### Option 4: Hybrid (Leaf + DPU VTEPs)

| Pros | Cons |
|------|------|
| Supports mixed environments | Two VTEP types to manage |
| Migration path from legacy | More complex troubleshooting |

### Option 5: Network-Centric (Leaf VTEP)

| Pros | Cons |
|------|------|
| Host is simple (VLANs only) | Doesn't leverage DPU capabilities |
| No BGP on hosts | Host needs bonding for redundancy |

### Option 6: Host OS Software VTEP (WORST)

| Pros | Cons |
|------|------|
| No special hardware needed | CPU overhead for encap/decap |
| | Competes with GPU workloads |
| | Higher latency |

## Where EVPN is Implemented in NCP Architecture

EVPN operates across three layers:

| Layer | EVPN Role |
|-------|-----------|
| **Converged Ethernet Fabric** | BGP EVPN control plane for VXLAN tenant isolation |
| **OOB Management Network** | Separate EVPN instance for IPMI and DPU management VXLANs |
| **Host-to-Fabric Connectivity** | EVPN-MH on leaves, DPU as EVPN VTEP |

## EVPN-MH vs. MC-LAG

NCP recommends **EVPN-MH** over MC-LAG:

| Aspect | MC-LAG (Not Recommended) | EVPN-MH (Recommended) |
|--------|--------------------------|------------------------|
| **State Sync** | Proprietary peer-link | Standards-based BGP |
| **Peer Link** | Required (wastes bandwidth) | Not required |
| **Scale** | Limited to 2 switches | Scales beyond 2 |
| **Host Config** | Requires LAG/bonding | No host-side config needed |
| **Standards** | Proprietary | RFC 7432 |
| **Failover** | Peer-link dependency | BGP route withdrawal |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVPN-MH (RECOMMENDED)                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│        ┌─────────┐                 ┌─────────┐                         │
│        │ Leaf 1  │◄───BGP EVPN────►│ Leaf 2  │                         │
│        │         │  (via Spines)   │         │                         │
│        │  ES: 1  │                 │  ES: 1  │  ← Same Ethernet Segment│
│        └────┬────┘                 └────┬────┘                         │
│             │                           │                               │
│             └───────────┬───────────────┘                               │
│                         │                                               │
│                    ┌────┴────┐                                          │
│                    │  Host   │                                          │
│                    │(no LAG!)│  ← No host-side config needed           │
│                    └─────────┘                                          │
│                                                                         │
│   • No peer link required                                              │
│   • Standards-based BGP state sync                                     │
│   • Active-active forwarding                                           │
│   • Works with DPU hardware ECMP                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## DPU as EVPN VTEP

In DPU Mode, the BlueField DPU participates directly in EVPN:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DPU AS EVPN VTEP                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                 HOST (x86)                                       │  │
│   │                                                                  │  │
│   │   Tenant VMs / Pods                                             │  │
│   │   ┌─────┐ ┌─────┐ ┌─────┐                                      │  │
│   │   │ VM1 │ │ VM2 │ │ Pod │  ← See single NIC interface          │  │
│   │   └──┬──┘ └──┬──┘ └──┬──┘    No bonding config needed          │  │
│   │      │       │       │                                          │  │
│   └──────┼───────┼───────┼──────────────────────────────────────────┘  │
│          │       │       │                                              │
│   ┌──────┴───────┴───────┴──────────────────────────────────────────┐  │
│   │              BlueField-4 DPU                                     │  │
│   │                                                                  │  │
│   │   ┌────────────────────────────────────────────────┐            │  │
│   │   │           EVPN Control Plane (FRR)             │            │  │
│   │   │  • iBGP peers with spine RRs                   │            │  │
│   │   │  • Advertises VM/Pod MAC+IP → VNI              │            │  │
│   │   │  • Learns remote host reachability             │            │  │
│   │   └────────────────────────────────────────────────┘            │  │
│   │                         │                                        │  │
│   │                         ▼ Programs hardware                      │  │
│   │   ┌────────────────────────────────────────────────┐            │  │
│   │   │           ASAP² Data Plane (Hardware)          │            │  │
│   │   │  • VXLAN encap/decap at line rate              │            │  │
│   │   │  • Hardware ECMP across uplinks                │            │  │
│   │   │  • Per-flow load balancing + fast failover     │            │  │
│   │   └────────────────────────────────────────────────┘            │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   HOST OS: Sees single logical NIC, no LAG/bonding                     │
│   DPU: Handles EVPN, ECMP, failover—all in hardware                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Distributed Anycast Gateway

EVPN enables every leaf/DPU to be the default gateway for a subnet:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED ANYCAST GATEWAY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   All VTEPs share the SAME gateway IP and MAC:                         │
│   • Gateway IP: 10.0.1.1                                               │
│   • Gateway MAC: 00:00:5e:00:01:01                                     │
│                                                                         │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐                  │
│   │  DPU 1  │         │  DPU 2  │         │  DPU 3  │                  │
│   │  VTEP   │         │  VTEP   │         │  VTEP   │                  │
│   │         │         │         │         │         │                  │
│   │ SVI:    │         │ SVI:    │         │ SVI:    │                  │
│   │ 10.0.1.1│         │ 10.0.1.1│         │ 10.0.1.1│  ← Same IP!     │
│   └─────────┘         └─────────┘         └─────────┘                  │
│                                                                         │
│   Benefits:                                                            │
│   • No hairpinning to centralized gateway                              │
│   • First-hop routing at the local VTEP                                │
│   • Host mobility without gateway change                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## VXLAN-to-PKey Mapping (InfiniBand Consistency)

For clusters with InfiniBand compute fabric, EVPN maintains isolation consistency:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                VXLAN → PKEY MAPPING (via UFM API)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Common Networking (Ethernet)          Compute Fabric (InfiniBand)    │
│                                                                         │
│   Tenant A: VNI 10001  ─────────────►   PKey 0x8001                    │
│   Tenant B: VNI 20001  ─────────────►   PKey 0x8002                    │
│   Tenant C: VNI 30001  ─────────────►   PKey 0x8003                    │
│                                                                         │
│   Consistent tenant isolation across both fabrics                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Cumulus Linux / FRR Configuration Example

```bash
# === UNDERLAY BGP (eBGP) ===
nv set router bgp autonomous-system 65001
nv set router bgp router-id 10.0.0.1

# Peer with spines (underlay)
nv set vrf default router bgp neighbor 10.255.0.1 remote-as 65000
nv set vrf default router bgp neighbor 10.255.0.2 remote-as 65000

# === OVERLAY BGP EVPN (iBGP) ===
# Enable EVPN address family with spine RRs
nv set vrf default router bgp neighbor 10.255.0.1 address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor 10.255.0.2 address-family l2vpn-evpn enable on

# Enable EVPN and advertise VNIs
nv set evpn enable on
nv set vrf default router bgp address-family l2vpn-evpn enable on

# Configure VNI with route targets
nv set evpn vni 10001 route-target export 65000:10001
nv set evpn vni 10001 route-target import 65000:10001

# Distributed anycast gateway
nv set interface vlan100 ip address 10.0.1.1/24
nv set interface vlan100 ip vrr address 10.0.1.1/24
nv set interface vlan100 ip vrr mac-address 00:00:5e:00:01:01

nv config apply
```

## The Air Traffic Control Analogy

| Air Traffic Control | EVPN in NCP |
|---------------------|-------------|
| Tracks every aircraft position | Type-2 routes: tracks every MAC/IP and its VTEP |
| Coordinates multiple runways | Active-active ECMP across multiple links |
| Redirects traffic on runway closure | Fast convergence: BGP route withdrawal on failure |
| Keeps airlines separated | VNI isolation: tenants never cross paths |
| Ground control at every airport | Distributed anycast gateway at every VTEP |
| Central flight database | BGP route reflectors on spines |

## Key Takeaways

1. **Two BGP domains**: Underlay (eBGP, VTEP reachability) and Overlay (iBGP EVPN, MAC/IP→VTEP mappings)
2. **EVPN Type-2 routes** advertise MAC + IP + VNI + VTEP—the core of overlay learning
3. **Route reflectors** (typically spines) eliminate full-mesh iBGP between DPUs
4. **DPU as VTEP** is the recommended model—leaves become simple L3 routers
5. **EVPN-MH** replaces MC-LAG for host multihoming—standards-based, no peer-link
6. **Distributed anycast gateway** enables first-hop routing at every VTEP
7. **Host OS is unaware** of BGP/EVPN—DPU abstracts all overlay complexity
8. **Hardware acceleration** via ASAP² handles VXLAN encap/decap at line rate
