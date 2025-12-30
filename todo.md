# NCP Reference Architecture Concepts - Learning Path

## Progress Tracker

| # | Topic | Status |
|---|-------|--------|
| 1 | Physical Isolation (OOB Management) | [x] |
| 2 | VLAN Tagging | [x] |
| 3 | VXLAN (Virtual Extensible LAN) | [x] |
| 4 | BGP EVPN (Control Plane for VXLAN) | [x] |
| 5 | VRF (Virtual Routing and Forwarding) | [ ] |
| 6 | Partition Keys (PKey) - InfiniBand Mapping | [ ] |
| 7 | MC-LAG (Multi-Chassis Link Aggregation) | [ ] |
| 8 | EVPN-MH (Ethernet VPN Multihoming) | [ ] |
| 9 | Hardware-Accelerated ECMP | [ ] |
| 10 | VF-LAG (Virtual Function Link Aggregation) | [ ] |
| 11 | RoCEv2 (RDMA over Converged Ethernet) | [ ] |
| 12 | Hardware-Based Congestion Control | [ ] |
| 13 | Adaptive Routing | [ ] |
| 14 | Zero-Trust Tenant Isolation | [ ] |
| 15 | NVIDIA DOCA Microservices | [ ] |

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

### Phase 3: Traffic Management (Topics 11-13)
Quality of service and performance optimization

- **RoCEv2** - RDMA protocol for storage/HPC
- **Congestion Control** - Preventing interference between traffic types
- **Adaptive Routing** - Dynamic path selection for load balancing

### Phase 4: Security Framework (Topics 14-15)
DPU-centric security and management

- **Zero-Trust** - DPU-enforced tenant isolation
- **DOCA Microservices** - Unified management framework
