# VLAN Tagging (802.1Q)

## Core Concept

A VLAN (Virtual LAN) is a **logical broadcast domain** within a physical switch. VLAN tagging (IEEE 802.1Q) inserts a 4-byte tag into the Ethernet frame header containing a **12-bit VLAN ID** (4094 usable VLANs).

Traffic in one VLAN cannot reach another VLAN without passing through a router (inter-VLAN routing). This provides **Layer 2 isolation** on shared physical infrastructure.

## 802.1Q Frame Structure

```
┌──────────┬──────────┬────────────────┬──────────┬─────────┬───────┐
│ Dest MAC │ Src MAC  │   802.1Q Tag   │ EtherType│ Payload │  FCS  │
│  6 bytes │ 6 bytes  │    4 bytes     │ 2 bytes  │         │       │
└──────────┴──────────┴───────┬────────┴──────────┴─────────┴───────┘
                              │
                    ┌─────────┴─────────┐
                    │  TPID   │   TCI   │
                    │ 0x8100  │         │
                    └─────────┴────┬────┘
                                   │
                         ┌─────────┴─────────┐
                         │ PCP │DEI│ VLAN ID │
                         │ 3b  │1b │  12 bits│
                         └─────────┴─────────┘
```

| Field | Size | Purpose |
|-------|------|---------|
| TPID | 16 bits | Tag Protocol ID (0x8100 = 802.1Q) |
| PCP | 3 bits | Priority Code Point (QoS) |
| DEI | 1 bit | Drop Eligible Indicator |
| VLAN ID | 12 bits | VLAN identifier (0-4095) |

## Port Types

| Port Type | Behavior | Use Case |
|-----------|----------|----------|
| **Access** | Belongs to ONE VLAN. Frames leave untagged. Switch adds tag on ingress, strips on egress. | End hosts (servers, storage) |
| **Trunk** | Carries MULTIPLE VLANs. Frames stay tagged (except native VLAN). | Switch-to-switch links, hypervisors, DPUs |

**Native VLAN**: On a trunk port, one VLAN can be designated "native"—frames in that VLAN are sent untagged. Handles devices that don't understand 802.1Q.

## 1 VLAN = 1 IP Subnet Pattern

The common pattern maps each VLAN to a dedicated IP subnet:

```
┌──────┬─────────────────┬───────────────────────────────────────────────┐
│ VLAN │    IP Subnet    │                  Purpose                      │
├──────┼─────────────────┼───────────────────────────────────────────────┤
│  10  │ 10.10.0.0/24    │ Infrastructure management (switches, DPUs)    │
│  20  │ 10.20.0.0/24    │ Storage - Control plane (NFS mounts)          │
│  21  │ 10.21.0.0/24    │ Storage - Data plane (RoCEv2)                 │
│  30  │ 10.30.0.0/24    │ K8s Node network                              │
│  31  │ 10.31.0.0/16    │ K8s Pod network                               │
├──────┼─────────────────┼───────────────────────────────────────────────┤
│ 100  │ 10.100.0.0/24   │ Tenant A - Ingress                            │
│ 101  │ 10.101.0.0/24   │ Tenant A - Internal services                  │
│ 200  │ 10.200.0.0/24   │ Tenant B - Ingress                            │
│ 201  │ 10.201.0.0/24   │ Tenant B - Internal services                  │
└──────┴─────────────────┴───────────────────────────────────────────────┘
```

## Cumulus Linux Example

```bash
# Create VLANs and assign to a bridge
nv set bridge domain br_default vlan 100
nv set bridge domain br_default vlan 200

# Access port: swp1 in VLAN 100 only
nv set interface swp1 bridge domain br_default access 100

# Trunk port: swp49 carries VLANs 100 and 200
nv set interface swp49 bridge domain br_default vlan 100
nv set interface swp49 bridge domain br_default vlan 200

# Apply
nv config apply
```

## The 4094 VLAN Limit Problem

### Per-Tenant VLAN Requirements (Typical)

| Traffic Type | VLANs per Tenant |
|--------------|------------------|
| Ingress (inference API) | 1 |
| Internal services | 1-3 |
| Storage | 1-2 |
| K8s namespaces | 2-5 |
| **Total** | **5-11** |

### Scale Math

| Scenario | Tenants | VLANs/Tenant | Total VLANs | Verdict |
|----------|---------|--------------|-------------|---------|
| Small enterprise | 20 | 5 | 100 | OK |
| Medium enterprise | 100 | 8 | 800 | OK |
| Large enterprise | 500 | 8 | 4,000 | Tight |
| Cloud provider | 1,000 | 5 | 5,000 | **Exceeded** |

### Additional Pressure

- **Multi-DC**: Same VLAN IDs needed across data centers = collision risk
- **Acquisitions**: Merging companies with overlapping VLAN schemes
- **Microsegmentation**: Zero-trust requires isolating individual workloads

## Common Pattern: VLANs for Infrastructure, VXLAN for Tenants

```
┌─────────────────────────────────────────────────────────────────────────┐
│   VLANs (4094 max)              VXLAN VNIs (16 million)                │
├─────────────────────────────────────────────────────────────────────────┤
│   • Infrastructure traffic      • Tenant isolation                      │
│   • Storage (RoCEv2 - no        • Per-tenant networks                   │
│     encap overhead)             • K8s namespace isolation               │
│   • Management                  • Workload microsegmentation            │
│   • VXLAN underlay transport    • Anything spanning L3 boundaries       │
│                                                                         │
│   Fixed, well-defined           Dynamic, scales with tenants            │
│   Configured on physical        Overlay - managed by EVPN               │
│   switches                                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

## Limitations Addressed by VXLAN

| VLAN Limitation | VXLAN Solution |
|-----------------|----------------|
| 4094 VLAN limit | 24-bit VNI = 16 million segments |
| L2 boundary (doesn't span routers) | Encapsulates L2 over L3 UDP |
| Spanning Tree required for redundancy | Uses IP routing (ECMP) for multipathing |

## Key Takeaways

1. **VLANs** provide Layer 2 isolation via 802.1Q tagging on shared physical switches
2. **1 VLAN = 1 IP subnet** is the common mapping pattern
3. **Access ports** serve end hosts (untagged); **trunk ports** carry multiple VLANs between switches
4. **4094 VLAN limit** becomes a problem at scale with multi-tenancy
5. **VLANs handle infrastructure traffic** (storage, management, VXLAN underlay); **VXLAN handles tenant traffic**
6. VLANs are the **underlay** that carries VXLAN-encapsulated traffic across the physical fabric
