# Hardware-Accelerated ECMP

## Core Concept

**ECMP = "Multiple equal-cost paths. Hash determines which path."**

When a router has multiple routes to the same destination with equal cost, it can use all of them simultaneously. A hash of the flow (5-tuple) determines which path each flow takes.

**Hardware-accelerated** = the hash + forwarding happens in DPU silicon at line rate, not in software.

```
DPU routing table:
  10.0.0.0/8 via 169.254.0.1 (Leaf 1)  cost 10  ┐
  10.0.0.0/8 via 169.254.0.3 (Leaf 2)  cost 10  ┘ ECMP group

Flow A (hash=0x3F) → Leaf 1
Flow B (hash=0x7A) → Leaf 2
Flow C (hash=0x12) → Leaf 1
```

## ECMP in DPU Mode

This is the NCP-recommended pattern for DPU-centric deployments:

```
┌─────────────────────────────────────────────────────────────┐
│                        DPU                                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              FRRouting (BGP)                        │   │
│  │  Learns routes from both leaves                     │   │
│  │  Installs ECMP group in hardware FIB                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  eth0: 169.254.0.1/31 ────► Leaf 1                         │
│  eth1: 169.254.0.3/31 ────► Leaf 2                         │
│                                                             │
│  Hardware: ConnectX/BlueField silicon                       │
│  • Per-flow hash at line rate                              │
│  • Failover in microseconds                                │
└─────────────────────────────────────────────────────────────┘
```

- DPU runs BGP, peers with both leaves
- Both leaves advertise same routes (default, fabric prefixes)
- DPU installs ECMP group in hardware
- Flows are hashed across both uplinks

## Why "Hardware-Accelerated" Matters

| Aspect | Software ECMP | Hardware ECMP (DPU) |
|--------|---------------|---------------------|
| **Hash location** | CPU | ASIC |
| **Throughput** | Limited by CPU | Line rate (400G+) |
| **Latency** | Variable | Deterministic, low |
| **Failover** | Milliseconds | Microseconds |
| **CPU overhead** | Yes | None for data plane |

## ECMP vs EVPN-MH

| Scenario | Technique | Why |
|----------|-----------|-----|
| **DPU is VTEP** (DPU Mode) | L3 ECMP | DPU routes, L3 uplinks |
| **Switch is VTEP** | EVPN-MH | Host has L2 uplinks, switches coordinate |

```
ECMP (DPU Mode)                    EVPN-MH (Switch Mode)
───────────────                    ─────────────────────
DPU routes                         Switches coordinate
L3 uplinks (two IPs)               L2 uplinks (bridge, one IP)
No switch EVPN-MH config           Requires switch EVPN-MH config
DPU runs BGP                       DPU uses anycast gateway
```

See [EVPN-MH](08-evpn-mh.md) for the two dual-homing options comparison.

## Hash Behavior

| Aspect | Behavior |
|--------|----------|
| **Input** | 5-tuple: src IP, dst IP, src port, dst port, protocol |
| **Granularity** | Per-flow (not per-packet) |
| **Persistence** | Same flow always takes same path (until path fails) |
| **Rebalancing** | Only on path add/remove |

Per-flow hashing ensures TCP connections stay on one path, avoiding out-of-order packets.

## Failover Behavior

```
Normal operation:
  Flow A → Leaf 1
  Flow B → Leaf 2

Leaf 1 link fails:
  Flow A → Leaf 2  ← rehashed in microseconds
  Flow B → Leaf 2

Leaf 1 recovers:
  Flow A → Leaf 1  ← may rehash back
  Flow B → Leaf 2
```

Hardware detects link failure and updates ECMP group without CPU involvement.

## Benefits for AI Factory

1. **No host bonding** — Host OS sees single logical interface; DPU handles redundancy
2. **Simplified switches** — Leaves just route; no EVPN-MH coordination needed
3. **Line-rate failover** — Hardware detects link down, redirects in microseconds
4. **Per-flow load balancing** — Distributes traffic across both uplinks
5. **Standard BGP** — No proprietary protocols; FRRouting on DPU

## BGP Route Advertisement Patterns

When DPU runs BGP for Kubernetes, two patterns exist:

### Pattern 1: Aggregate Prefix per Node (Pod CIDR)

```
Node 1 announces: 10.244.1.0/24 (pod CIDR)
Node 2 announces: 10.244.2.0/24 (pod CIDR)
```

- Used for **pod networking** (direct pod IP routing)
- Safe because pods are always local to the node
- Calico, Cilium in BGP mode use this

### Pattern 2: /32 per Service IP (Service Advertisement)

```
Node 1 announces: 10.96.0.10/32 (CoreDNS)
Node 1 announces: 10.96.0.53/32 (my-app-svc)
Node 2 announces: 10.96.0.80/32 (other-svc)
```

- Used for **LoadBalancer / ExternalIP services**
- Only announced when service has healthy endpoints
- Withdrawn when service deleted or unhealthy
- **Standard pattern** for service IPs—avoids blackholing

### Why /32 for Services?

| Concern | Aggregate (/22) | Per-service (/32) |
|---------|-----------------|-------------------|
| **Blackhole risk** | High | None |
| **BGP churn** | Low | Higher (acceptable) |
| **Health tracking** | Hard | Easy—withdraw on failure |
| **Service mobility** | Breaks | Works |

### CNI Behavior

| CNI | Pod CIDR | Service IPs |
|-----|----------|-------------|
| **Calico** | /24-/26 per node | /32 per service |
| **Cilium** | /24 per node | /32 per service |
| **MetalLB** | N/A | /32 per LoadBalancer |

```
kube-apiserver
     │
     │ watch Services
     ▼
┌─────────────┐
│ BGP Speaker │ (MetalLB, Cilium, Calico)
│ on DPU/node │
└──────┬──────┘
       │ announce/withdraw /32
       ▼
   Leaf Switch
```

## Cumulus Linux Configuration (Leaf Side)

```bash
# BGP peering with DPU
nv set vrf default router bgp neighbor 169.254.0.0 remote-as external
nv set vrf default router bgp neighbor 169.254.0.2 remote-as external

# Enable ECMP (multiple paths)
nv set vrf default router bgp address-family ipv4-unicast maximum-paths 64

# Advertise default route to DPUs
nv set vrf default router bgp address-family ipv4-unicast network 0.0.0.0/0

nv config apply
```

## DPU Configuration (FRRouting)

```bash
# /etc/frr/frr.conf on DPU
router bgp 65100
  neighbor 169.254.0.1 remote-as external
  neighbor 169.254.0.3 remote-as external

  address-family ipv4 unicast
    maximum-paths 2
    network 10.244.1.0/24  # Pod CIDR for this node
  exit-address-family
```

## Key Takeaways

1. **ECMP = "Multiple equal-cost paths, hash picks one"** — Per-flow load balancing
2. **Hardware-accelerated** — DPU ASIC does hashing at line rate, microsecond failover
3. **DPU Mode pattern** — DPU runs BGP, L3 uplinks to leaves, no EVPN-MH needed
4. **Alternative to EVPN-MH** — For DPU-centric deployments where DPU is VTEP
5. **Pod CIDR** — Announce aggregate (/24) per node, safe because pods are local
6. **Service IPs** — Announce /32 per service, withdraw on failure, avoids blackholing
7. **No host bonding** — DPU handles redundancy; host sees single interface
