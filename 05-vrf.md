# VRF (Virtual Routing and Forwarding)

## Core Concept

**VRF = "Which FIB to use?" Determined by ingress interface.**

A VRF creates isolated routing tables on a single device. Traffic arriving on an interface assigned to VRF-A uses VRF-A's FIB. Traffic on VRF-B interfaces uses VRF-B's FIB.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SINGLE SWITCH/DPU                             │
│                                                                         │
│   ┌─────────────────┐              ┌─────────────────┐                  │
│   │    VRF: red     │              │   VRF: blue     │                  │
│   │                 │              │                 │                  │
│   │  FIB:           │              │  FIB:           │                  │
│   │  10.0.0.0/24    │              │  10.0.0.0/24    │  ← Same range,   │
│   │  → next-hop A   │              │  → next-hop B   │    different FIB │
│   │                 │              │                 │                  │
│   │  Interfaces:    │              │  Interfaces:    │                  │
│   │  swp1, swp2     │              │  swp3, swp4     │                  │
│   └─────────────────┘              └─────────────────┘                  │
│                                                                         │
│   Traffic in swp1 → uses red FIB                                        │
│   Traffic in swp3 → uses blue FIB                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Rules

1. Each interface belongs to **exactly one** VRF
2. Each VRF has its **own FIB** (forwarding table)
3. VRFs are **isolated**—no route leaking unless explicitly configured
4. Same IP ranges can exist in different VRFs without conflict

## Two Multitenancy Patterns

### Pattern 1: Port-Based Multitenancy (VRF)

**"These ports belong to Tenant A, those ports to Tenant B"**

```
┌─────────────────────────────────────────────────────────────┐
│                      SWITCH                                 │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │  VRF: tenant-a  │         │  VRF: tenant-b  │           │
│  │  Ports 1-4      │         │  Ports 5-8      │           │
│  │  Dedicated      │         │  Dedicated      │           │
│  └─────────────────┘         └─────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

- Physical port assignment determines tenant
- Dedicated capacity per tenant
- Tenant changes require rewiring

### Pattern 2: Overlay Multitenancy (VXLAN/VNI)

**"Fabric is shared. VNI determines tenant."**

```
┌─────────────────────────────────────────────────────────────┐
│                      SWITCH                                 │
│                   All ports shared                          │
│  ┌────────────────────────────────────────────────────────┐│
│  │  VNI 10001 → Tenant A traffic                          ││
│  │  VNI 20001 → Tenant B traffic                          ││
│  │  VNI 30001 → Tenant C traffic                          ││
│  │  (all on same physical ports)                          ││
│  └────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

- Same ports carry all tenants
- Tenant defined by VM/pod assignment → VNI tagging
- QoS enforces fair bandwidth sharing
- Software-defined, no rewiring

### Pattern Comparison

| Aspect | Pattern 1: VRF (Port-based) | Pattern 2: VXLAN (Overlay) |
|--------|----------------------------|---------------------------|
| **Isolation by** | Physical port | VNI tag |
| **Capacity model** | Dedicated | Shared + QoS quotas |
| **Tenant assignment** | Port → Tenant | VM/Pod → VNI → Tenant |
| **Flexibility** | Low (rewire to change) | High (software-defined) |
| **Oversubscription** | None (dedicated) | Yes (statistical sharing) |
| **Use case** | Colo, bare-metal, enterprise | Cloud, K8s, AI inference |

### AI Factory: Which Pattern?

| Workload | Pattern | Why |
|----------|---------|-----|
| **Multi-tenant inference (K8s)** | VXLAN overlay | Pods dynamically scheduled, shared GPUs, VNI isolates |
| **Dedicated training (Slurm)** | VRF or none | Single tenant, no multi-tenancy needed |
| **Infrastructure separation** | VRF | Isolate storage, mgmt, compute domains |

**Hybrid is common**: VXLAN for tenant workloads + VRF for infrastructure domains.

## VRF Use Cases in AI Factory

### 1. Infrastructure vs Tenant Separation

```
VRF: infrastructure                VRF: tenants
├─ Interfaces: mgmt ports          ├─ Interfaces: workload ports
├─ Routes to:                      ├─ Routes to:
│  • K8s control plane             │  • Tenant workloads
│  • Monitoring (Prometheus)       │  • External APIs
│  • Storage control plane         │  • NO route to infra!
```

Even if a tenant pod is compromised, it has no route to infrastructure.

### 2. Storage Traffic Isolation

```
VRF: storage                       VRF: compute
├─ Interfaces: storage NICs        ├─ Interfaces: inference NICs
├─ Routes to:                      ├─ Routes to:
│  • Storage arrays                │  • Tenant traffic
│  • Lossless, dedicated BW        │  • K8s services
│  • No external routes            │  • External egress
```

Storage VRF carries latency-sensitive RoCEv2. Compute traffic can't congest storage.

### 3. Overlapping Tenant IPs (with VXLAN)

When combined with VXLAN, VRF enables overlapping IPs across tenants:

```
VRF: tenant-a (L3VNI 3001)         VRF: tenant-b (L3VNI 3002)
├─ Subnets:                        ├─ Subnets:
│  10.0.0.0/24 (VNI 10001)         │  10.0.0.0/24 (VNI 20001)  ← Same!
│  10.0.1.0/24 (VNI 10002)         │  10.0.1.0/24 (VNI 20002)  ← Same!
```

Each tenant has their own VRF with their own L3VNI for inter-subnet routing.

## VRF + VXLAN: L3VNI

In an EVPN/VXLAN fabric, each VRF is assigned an **L3VNI** for inter-VTEP routing:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TENANT A                                        │
│                                                                         │
│   VNI 10001 (L2)          VNI 10002 (L2)                               │
│   Subnet: 10.1.0.0/24     Subnet: 10.2.0.0/24                          │
│        │                       │                                        │
│        └───────────┬───────────┘                                        │
│                    │                                                    │
│              ┌─────┴─────┐                                              │
│              │  VRF-A    │  Routes between subnets within tenant       │
│              │ L3VNI 3001│  L3VNI carries routed traffic across fabric │
│              └───────────┘                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

- **L2VNI**: Carries bridged traffic within a single subnet
- **L3VNI**: Carries routed traffic between subnets in the same VRF

## Distributed Anycast Gateway

With VRF, gateways are **distributed** across all leaves (or DPUs):

```
Leaf 1                           Leaf 2
┌──────────────────┐             ┌──────────────────┐
│ VRF: tenant-a    │             │ VRF: tenant-a    │
│ SVI: 10.1.0.1    │             │ SVI: 10.1.0.1    │  ← Same IP
│ MAC: aa:aa:aa    │             │ MAC: aa:aa:aa    │  ← Same MAC
└──────────────────┘             └──────────────────┘
```

- Same gateway IP/MAC on every leaf/DPU (anycast)
- Host always reaches gateway in one hop
- No hairpin through centralized router

## VRF in DPU Mode

When the DPU is the VTEP, VRFs live on the DPU:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           HOST                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │
│  │ Tenant A VM │  │ Tenant B VM │  │ Storage     │                     │
│  │ 10.0.0.5    │  │ 10.0.0.5    │  │ RoCEv2 NIC  │                     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                     │
│         │                │                │                             │
│  ┌──────┴────────────────┴────────────────┴──────┐                     │
│  │              BlueField DPU                    │                     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │                     │
│  │  │VRF:      │  │VRF:      │  │VRF:      │    │                     │
│  │  │tenant-a  │  │tenant-b  │  │storage   │    │                     │
│  │  │L3VNI 3001│  │L3VNI 3002│  │L3VNI 3003│    │                     │
│  │  └──────────┘  └──────────┘  └──────────┘    │                     │
│  └───────────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

The DPU provides per-tenant VRF isolation at the host edge.

## VRF for Storage: NCP Guidance

From NCP reference: **VLAN tagging is preferred for storage** over VRF.

Why?
- Storage traffic (RoCEv2) is latency-sensitive
- VRF adds routing lookup overhead
- Storage typically doesn't need overlapping IPs
- Simpler = better for dedicated storage networks

```
┌─────────────────────────────────────────┐
│ VRF-based isolation                     │
│ • Multi-tenant workloads                │
│ • Overlapping IPs needed                │
│ • Complex routing policies              │
├─────────────────────────────────────────┤
│ VLAN-based isolation (storage)          │
│ • Dedicated storage VLAN                │
│ • Single flat subnet                    │
│ • No routing overhead                   │
│ • RoCEv2 lossless traffic               │
└─────────────────────────────────────────┘
```

## Cumulus Linux Configuration

```bash
# Create VRF
nv set vrf tenant-a

# Assign L3VNI to VRF (for EVPN routing)
nv set vrf tenant-a evpn vni 3001

# Create SVI (gateway) in VRF
nv set interface vlan100 ip address 10.1.0.1/24
nv set interface vlan100 ip vrf tenant-a

# Map L2VNI to VLAN
nv set bridge domain br_default vlan 100 vni 10001

# Assign physical interface to VRF (for port-based pattern)
nv set interface swp1 ip vrf tenant-a

nv config apply
```

## Key Takeaways

1. **VRF = "Which FIB?"** — Ingress interface determines which routing table is used
2. **Two multitenancy patterns**: Port-based (VRF) for dedicated, Overlay (VXLAN) for shared
3. **AI Factory hybrid**: VXLAN for tenant workloads + VRF for infrastructure domains
4. **Enables overlapping IPs** — Different tenants can use same IP ranges
5. **L3VNI** — Each VRF gets an L3VNI for inter-subnet routing across VXLAN fabric
6. **Anycast gateway** — Same gateway IP/MAC on every leaf/DPU in the VRF
7. **DPU Mode** — VRFs live on the DPU, providing per-tenant isolation at host edge
8. **Storage exception** — NCP recommends VLANs (not VRF) for storage traffic simplicity
