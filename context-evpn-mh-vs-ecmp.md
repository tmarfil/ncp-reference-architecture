# NCP Reference: EVPN-MH vs L3 ECMP

## Summary

The NCP Reference Architecture recommends **both** approaches for different scenarios:

| Deployment Model | Recommended Technique | VTEP Location |
|------------------|----------------------|---------------|
| Standard (switch-centric) | EVPN-MH | Leaf switches |
| DPU Mode / Zero Trust | L3 ECMP | BlueField DPU |

---

## EVPN-MH: Baseline Recommendation

**Status**: Tested and recommended as part of the reference architecture.

**When**: Switch-centric deployments where leaves are VTEPs.

**Why**:
- Standards-based (BGP control plane for state sync and loop avoidance)
- Active-active forwarding on all host links
- No dedicated peer-links required
- Near-seamless failover during link/node failures

**Explicitly NOT recommended**: MC-LAG (proprietary, complex, inefficient)

**Requires**:
- EVPN-MH configuration on leaf switches
- Host-side bonding/aggregation (bridge interface)

---

## L3 ECMP: DPU Mode Preference

**Status**: Key benefit for DPU-centric clusters.

**When**: BlueField DPUs operating in "DPU Mode" or "DPU Mode Zero Trust."

**How it works**:
- Each DPU acts as L3 router and EVPN VTEP
- DPU participates directly in fabric control plane
- Per-flow ECMP load balancing handled in DPU hardware

**Benefits**:
- **Simplified switches**: No EVPN-MH config on leaves; basic BGP/EVPN peering only
- **No host bonding**: Host OS sees single NIC; DPU manages redundancy
- **Hardware offload**: Failover and load balancing entirely in DPU silicon

---

## Comparison Table

| Aspect | EVPN-MH | L3 ECMP (DPU Mode) |
|--------|---------|-------------------|
| **Document status** | Tested and recommended baseline | Key benefit for DPU clusters |
| **VTEP location** | Leaf switches | DPU |
| **Switch config** | EVPN-MH + BGP state sync | Basic BGP/EVPN peering |
| **Host config** | Bonding/bridge required | No bonding required |
| **Where intelligence lives** | Switches | DPU |

---

## Conceptual Positioning

- **EVPN-MH** = "Gold standard" for standard network resiliency
- **L3 ECMP with DPU Mode** = "Advanced evolution" for AI factories that move intelligence into the DPU to simplify surrounding network infrastructure
