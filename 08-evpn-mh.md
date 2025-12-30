# EVPN-MH (Ethernet VPN Multihoming)

## Core Concept

EVPN-MH provides **switch-side coordination** for hosts connected to multiple leaf switches, enabling redundancy and load balancing with minimal host configuration (bridge, no LACP/bonding driver).

The key insight: the intelligence lives in the switches, not the host.

## When to Use EVPN-MH vs L3 ECMP

The NCP Reference Architecture recommends different techniques based on VTEP location:

| Deployment Model | VTEP Location | Recommended Technique |
|------------------|---------------|----------------------|
| **Switch-centric** | Leaf switches | EVPN-MH |
| **DPU-centric (DPU Mode)** | BlueField DPU | L3 ECMP |

```
Switch-Centric (EVPN-MH)              DPU-Centric (L3 ECMP)
────────────────────────              ────────────────────
     Host                                  Host OS
   (bridge)                              (single NIC view)
   ┌──┴──┐                                    │
   │     │                              ┌─────┴─────┐
   │     │                              │    DPU    │ ← VTEP here
┌──┴──┐ ┌┴────┐                         │ (L3 ECMP) │
│Leaf1│ │Leaf2│ ← VTEPs here            └─────┬─────┘
│EVPN-│ │EVPN-│                          ┌────┴────┐
│ MH  │ │ MH  │                          │         │
└─────┘ └─────┘                      ┌───┴──┐  ┌───┴──┐
                                     │Leaf1 │  │Leaf2 │ ← Just route
                                     └──────┘  └──────┘   underlay
```

**EVPN-MH** = "Gold standard" baseline for switch-centric deployments.
**L3 ECMP** = "Advanced evolution" for DPU-centric AI factories—moves intelligence into DPU, simplifies switch config.

## Two Options for Dual-Homed DPU Connectivity

### Option 1: EVPN-MH

```
DPU
├─ br0: 10.0.0.5              ← Single IP on bridge
│  ├─ eth0 ──► Leaf1 (EVPN-MH)
│  └─ eth1 ──► Leaf2 (EVPN-MH)
│
└─ default via 10.0.0.1       ← Anycast gateway (exists on both leaves)
```

- **DPU config**: Bridge interface across two links, single IP, forwards to anycast gateway
- **Switch config**: EVPN-MH, switches are VTEPs
- **Uplinks**: L2

### Option 2: L3 ECMP

```
DPU
├─ eth0: 169.254.0.1/31 ──► Leaf1    ← Separate L3 link
└─ eth1: 169.254.0.3/31 ──► Leaf2    ← Separate L3 link

BGP: ECMP routes to remote VTEPs via either leaf
```

- **DPU config**: Two separate interfaces, two IPs, runs BGP, ECMP routes
- **Switch config**: Basic BGP peering, just route underlay
- **Uplinks**: L3

### Comparison

| Aspect | Option 1: EVPN-MH | Option 2: L3 ECMP |
|--------|-------------------|-------------------|
| **DPU interface** | Bridge (L2) | Separate L3 links |
| **DPU IP addresses** | 1 (on bridge) | 2 (one per link) |
| **DPU runs BGP?** | No (uses gateway) | Yes |
| **Switch config** | EVPN-MH required | Basic BGP only |
| **VTEP location** | Leaf switches | DPU |

Both are valid. NCP recommends **Option 2** for DPU Mode deployments.

## Why EVPN-MH Exists

Traditional approaches have limitations:

| Approach | Limitation |
|----------|------------|
| **L2 LAG (LACP)** | All links must terminate on the **same switch**—single point of failure |
| **MC-LAG** | Proprietary, complex, inter-chassis link wastes ports |
| **L3 ECMP** | Requires host to run routing protocol (BGP, OSPF) |

EVPN-MH solves these: host connects to **different switches** with **no special host configuration**.

## Comparison: LAG vs L3 ECMP vs EVPN-MH

```
L2 LAG (LACP)                    L3 ECMP                      EVPN-MH
─────────────                    ───────                      ───────
     Host                           Host                         Host
   (bonded)                    (runs routing)               (no config!)
      │                         ┌───┴───┐                    ┌───┴───┐
      │                         │       │                    │       │
 ┌────┴────┐                    │       │                    │       │
 │ Single  │               ┌────┴──┐ ┌──┴────┐          ┌────┴──┐ ┌──┴────┐
 │ Switch  │               │Leaf 1 │ │Leaf 2 │          │Leaf 1 │ │Leaf 2 │
 └─────────┘               └───────┘ └───────┘          │ ES-ID │ │ ES-ID │
                                                        └───────┘ └───────┘
Single point               Host must run                 Switches
of failure                 BGP/OSPF                      coordinate
```

## How EVPN-MH Works

### Ethernet Segment (ES)

An ES is a **logical grouping** of links connecting to the same host. Switches identify ES membership via:

- **ES-ID**: Unique identifier for the segment (derived from `local-id`)
- **ES-sys-mac**: Shared MAC address (must match on all participating switches)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ETHERNET SEGMENT                                 │
│                                                                         │
│    Leaf 1                                      Leaf 2                   │
│    ┌──────────────────┐                ┌──────────────────┐             │
│    │ ES-ID: 1         │                │ ES-ID: 1         │ ◄── Match! │
│    │ ES-mac: aa:bb:cc │                │ ES-mac: aa:bb:cc │ ◄── Match! │
│    │ swp1 ────────────┼────────────────┼───────────► swp1 │             │
│    └──────────────────┘       │        └──────────────────┘             │
│                               │                                         │
│                               ▼                                         │
│                        ┌────────────┐                                   │
│                        │    Host    │                                   │
│                        │  eth0/eth1 │                                   │
│                        └────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### EVPN Route Types for Multihoming

| Type | Name | Purpose |
|------|------|---------|
| **Type-1** | Ethernet Auto-Discovery | Advertises ES membership; enables **aliasing** (remote VTEPs learn both paths) |
| **Type-4** | Ethernet Segment | DF election; prevents duplicate BUM delivery |

### Traffic Forwarding

| Traffic Type | Mode | Mechanism |
|--------------|------|-----------|
| **Unicast** | Active-Active | Both leaves forward; remote VTEP hashes flows across paths |
| **BUM** | Active-Standby | Only Designated Forwarder (DF) sends to host |

```
Unicast Load Balancing (at remote VTEP):

  Remote VTEP sees via Type-1 routes:
    "MAC 00:11:22:33:44:55 reachable via Leaf1 AND Leaf2"

  Hash decision:
    Flow A (hash=0x3F) → Leaf 1
    Flow B (hash=0x7A) → Leaf 2
    Flow C (hash=0x12) → Leaf 1
```

### Designated Forwarder (DF) Election

For BUM traffic, one leaf is elected DF per VLAN to prevent duplicates:

```
Broadcast from fabric to host:

    ┌─────────┐         ┌─────────┐
    │ Leaf 1  │         │ Leaf 2  │
    │  (DF)   │         │(Backup) │
    └────┬────┘         └────┬────┘
         │                   │
    Forwards BUM        Blocks BUM
         │              (would duplicate)
         ▼
    ┌─────────────────────────┐
    │         Host            │
    └─────────────────────────┘
```

## Linux Host Configuration

The host requires **no bonding**—just a bridge to aggregate interfaces:

```bash
# Create bridge
ip link add br0 type bridge
ip link set br0 up

# Add both interfaces (no LACP, no bonding driver)
ip link set eth0 up
ip link set eth1 up
ip link set eth0 master br0
ip link set eth1 master br0

# IP address on the bridge
ip addr add 10.100.1.10/24 dev br0
ip route add default via 10.100.1.1
```

```
┌─────────────────────────────────────────────────────────────┐
│                     LINUX HOST                              │
│                                                             │
│                    ┌─────────┐                              │
│                    │   br0   │ ◄── IP lives here            │
│                    │10.100.1.10                             │
│                    └────┬────┘                              │
│               ┌─────────┴─────────┐                         │
│           ┌───┴───┐           ┌───┴───┐                     │
│           │ eth0  │           │ eth1  │  No IPs, no bonding │
│           └───┬───┘           └───┬───┘                     │
└───────────────┼───────────────────┼─────────────────────────┘
                │                   │
                ▼                   ▼
           ┌─────────┐         ┌─────────┐
           │ Leaf 1  │         │ Leaf 2  │
           └─────────┘         └─────────┘
```

## Cumulus Linux Switch Configuration

### Leaf 1

```bash
# Define Ethernet Segment - MUST match Leaf 2
nv set interface swp1 evpn multihoming segment local-id 1
nv set interface swp1 evpn multihoming segment mac aa:bb:cc:dd:ee:01

# DF preference (higher = preferred for BUM)
nv set interface swp1 evpn multihoming segment df-preference 50000

# Standard access port
nv set interface swp1 bridge domain br_default access 100

nv config apply
```

### Leaf 2

```bash
# SAME ES-ID and ES-sys-mac as Leaf 1
nv set interface swp1 evpn multihoming segment local-id 1
nv set interface swp1 evpn multihoming segment mac aa:bb:cc:dd:ee:01

# Lower DF preference = backup for BUM
nv set interface swp1 evpn multihoming segment df-preference 32767

# Standard access port
nv set interface swp1 bridge domain br_default access 100

nv config apply
```

### Key Configuration Elements

| Element | Purpose | Must Match? |
|---------|---------|-------------|
| `local-id` | Creates ES-ID | Yes |
| `mac` | ES-sys-mac for coordination | Yes |
| `df-preference` | DF election weight | No (should differ) |

## Load Balancing Details

| Aspect | Behavior |
|--------|----------|
| **Layer** | L2 decision using L3/L4 flow hash |
| **Hash** | 5-tuple (src/dst IP, src/dst port, protocol) |
| **Granularity** | Per-flow (not per-packet) |
| **Location** | Ingress VTEP (remote switch choosing path) |

## Adaptive Routing and EVPN-MH

These operate at different layers:

| Feature | Scope | What It Optimizes |
|---------|-------|-------------------|
| **Adaptive Routing** | Underlay | Paths *through* spine (congestion-aware) |
| **EVPN-MH** | Overlay | Paths *to* ES members (hash-based) |

Adaptive routing helps traffic traverse the spine efficiently. EVPN-MH ES member selection remains hash-based, not congestion-aware.

## Scale Limits

| Limit | Value |
|-------|-------|
| Leaves per ES (typical) | 2-4 |
| Leaves per ES (Cumulus tested) | 4 |
| Theoretical (RFC) | 128 |

Two leaves per ES is the common deployment model—matches physical topology and provides adequate redundancy.

## EVPN-MH vs MC-LAG

| Aspect | EVPN-MH | MC-LAG |
|--------|---------|--------|
| **Standard** | IETF RFC 7432 | Proprietary per vendor |
| **Inter-switch link** | Not required | Required (wastes ports) |
| **Scale** | Better (no peer link bottleneck) | Limited by peer link |
| **Host config** | None (or simple bridge) | LACP bond required |
| **Recommendation** | Preferred | Legacy, avoid if possible |

## When EVPN-MH Applies

EVPN-MH applies to **switch-centric deployments** where leaves are VTEPs:

```
✓ EVPN-MH: Switch-centric (leaves are VTEPs)

    Host (bridge) ──┬── Leaf 1 (VTEP, EVPN-MH)
                    └── Leaf 2 (VTEP, EVPN-MH)

✗ NOT EVPN-MH: DPU-centric (DPU is VTEP)

    DPU (VTEP, L3) ──┬── Leaf 1 (routes underlay)
                     └── Leaf 2 (routes underlay)
```

For **DPU Mode** deployments where the BlueField DPU is the VTEP:
- DPU has L3 uplinks to leaves
- DPU uses hardware-accelerated ECMP for redundancy
- Leaves just route underlay—no EVPN-MH config needed
- See [Hardware-Accelerated ECMP](09-ecmp.md) for details

## Key Takeaways

1. **EVPN-MH** enables host multihoming across different switches with minimal host config (bridge)
2. **Switch-centric only**—applies when leaves are VTEPs; for DPU Mode, use L3 ECMP instead
3. **ES-ID + ES-sys-mac** must match on participating switches to form an Ethernet Segment
4. **Unicast = active-active** (flow-hashed); **BUM = active-standby** (DF election)
5. **Host configuration** is minimal—standard Linux bridge, IP on bridge interface
6. **Load balancing** is per-flow hash at the remote VTEP, not congestion-aware
7. **Preferred over MC-LAG**—standards-based, no wasted inter-switch ports
8. **Typical deployment** uses 2 leaves per ES, matching dual-NIC/DPU host topology
