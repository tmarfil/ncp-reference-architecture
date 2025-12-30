# NCP Reference Architecture Concepts - Learning Path

## Progress Tracker

| # | Topic | Status |
|---|-------|--------|
| 1 | Physical Isolation (OOB Management) | [x] |
| 2 | VLAN Tagging | [x] |
| 3 | VXLAN (Virtual Extensible LAN) | [x] |
| 4 | BGP EVPN (Control Plane for VXLAN) | [x] |
| 5 | VRF (Virtual Routing and Forwarding) | [x] |
| 6 | Partition Keys (PKey) - InfiniBand Mapping | [ ] |
| 7 | MC-LAG (Multi-Chassis Link Aggregation) | [ ] |
| 8 | EVPN-MH (Ethernet VPN Multihoming) | [x] |
| 9 | Hardware-Accelerated ECMP | [ ] |
| 10 | VF-LAG (Virtual Function Link Aggregation) | [ ] |
| 11 | RoCEv2 (RDMA over Converged Ethernet) | [ ] |
| 12 | DCB (Data Center Bridging) Overview | [ ] |
| 13 | PFC (Priority Flow Control) | [ ] |
| 14 | ETS (Enhanced Transmission Selection) | [ ] |
| 15 | DSCP/CoS Traffic Classification | [ ] |
| 16 | QoS Traffic Lanes for Converged Networking | [ ] |
| 17 | Hardware-Based Congestion Control | [ ] |
| 18 | Adaptive Routing | [ ] |
| 19 | Zero-Trust Tenant Isolation | [ ] |
| 20 | NVIDIA DOCA Microservices | [ ] |

---

## Learning Order Rationale

### Phase 1: Isolation Fundamentals (Topics 1-6)
Build understanding from physical → Layer 2 → Layer 3 → overlay isolation

- **Physical Isolation** - Simplest form; OOB management separation
- **VLAN** - Basic L2 segmentation within a switch/fabric
- **VXLAN** - Extends VLANs across L3 boundaries (overlay)
- **BGP EVPN** - Control plane that manages VXLAN state
- **VRF** - L3 routing table isolation per tenant
- **PKey** - How VXLAN maps to InfiniBand (optional if no IB)

### Phase 2: Host Connectivity & Resiliency (Topics 7-10)
How hosts connect redundantly to the fabric

- **MC-LAG** - Legacy approach (not recommended, but good context)
- **EVPN-MH** - Modern standards-based replacement
- **ECMP** - L3 multipathing via DPU
- **VF-LAG** - Hardware-accelerated bonding via OVS-DOCA

### Phase 3: Traffic Management & QoS (Topics 11-18)
Quality of service, lossless Ethernet, and performance optimization

- **RoCEv2** - RDMA protocol for storage/HPC (requires lossless Ethernet)
- **DCB Overview** - Data Center Bridging: umbrella for PFC + ETS + DCBx
- **PFC** - Priority Flow Control: per-priority PAUSE for lossless traffic classes
- **ETS** - Enhanced Transmission Selection: bandwidth guarantees per traffic class
- **DSCP/CoS Classification** - How traffic gets marked and queued
- **QoS Traffic Lanes** - Carving out dedicated lanes for storage, management, tenant traffic
- **Congestion Control** - ECN, DCQCN for RoCEv2; preventing interference
- **Adaptive Routing** - Dynamic path selection for load balancing

### Phase 4: Security Framework (Topics 19-20)
DPU-centric security and management

- **Zero-Trust** - DPU-enforced tenant isolation
- **DOCA Microservices** - Unified management framework

---

## QoS Traffic Lanes: Key Concept

Common Networking (North-South) carries multiple traffic types on a shared fabric. QoS ensures each gets appropriate treatment:

| Traffic Class | Priority | Lossless? | Bandwidth | Example DSCP |
|---------------|----------|-----------|-----------|--------------|
| **RoCEv2 Storage** | Highest | Yes (PFC) | Guaranteed (e.g., 50%) | 26 (AF31) |
| **In-band Management** | Medium | No | Guaranteed minimum | 16 (CS2) |
| **K8s / Inference Traffic** | Normal | No | Best-effort | 0 (BE) |
| **Tenant Ingress/Egress** | Normal | No | Best-effort | 0 (BE) |

The goal: RoCEv2 storage traffic gets **lossless, guaranteed bandwidth** while coexisting with other north-south traffic types.

**Note:** GPU/NCCL collective traffic travels on the dedicated East-West fabric (not covered here).
