# Appendix: DOCA Reference - Standard Linux vs. Hardware-Accelerated Components

## Overview

NVIDIA DOCA (Data Center Infrastructure On a Chip Architecture) unlocks hardware acceleration on BlueField DPUs. This appendix documents which components are standard Linux (portable to any ARM system) versus DOCA-specific (requiring BlueField hardware).

**Key Insight**: Control plane components (BGP, EVPN, routing daemons) are standard Linux software. Data plane components (packet forwarding, encapsulation, firewalling) benefit from DOCA hardware acceleration.

## Standard Linux vs. DOCA-Accelerated Components

| Category | Component | Standard Linux | DOCA-Accelerated | Notes | Documentation |
|----------|-----------|----------------|------------------|-------|---------------|
| **Switching** | Open vSwitch | OVS-Kernel (`apt install openvswitch-switch`) | **OVS-DOCA** (ASAP²) | Same CLI, hardware datapath | [OVS in DOCA](https://docs.nvidia.com/doca/sdk/openvswitch-acceleration---ovs-in-doca/index.html) |
| **Switching** | Traffic Control | TC flower (kernel) | **TC offload to eSwitch** | `skip_sw` flag for HW-only | [OVS-Kernel HW Offloads](https://docs.nvidia.com/doca/archive/2-7-0/ovs-kernel+hardware+offloads/index.html) |
| **Switching** | Flow Programming | TC / OVS flows | **DOCA Flow API** | Native API for eSwitch | [DOCA Switching](https://docs.nvidia.com/doca/sdk/doca-switching/index.html) |
| **Overlay** | VXLAN Tunnel | Kernel (`ip link add type vxlan`) | **OVS-DOCA VXLAN** | Line-rate encap/decap | [OVS Inside BlueField](https://docs.nvidia.com/doca/sdk/ovs+inside+bluefield/index.html) |
| **Routing** | BGP/OSPF/EVPN | FRRouting (`apt install frr`) | N/A - not needed | Control plane, any Linux | [FRRouting.org](https://frrouting.org/) |
| **Security** | Firewall / ACLs | iptables / nftables | **DOCA Firewall** | gRPC-based, containerized | [DOCA Firewall Guide](https://docs.nvidia.com/doca/archive/2-6-0/NVIDIA+DOCA+Firewall+Application+Guide/index.html) |
| **Security** | IPsec VPN | strongSwan (kernel crypto) | **DOCA IPsec** | Hardware crypto offload | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Security** | Runtime Inspection | N/A | **DOCA Argus** | AI runtime security, memory introspection | [DOCA Argus Guide](https://docs.nvidia.com/doca/archive/3-0-0-vgt-update/doca-argus-service-guide/index.html) |
| **Security** | Pattern Matching | YARA (CPU) | **DOCA YARA Inspection** | Hardware-accelerated scanning | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Storage** | NVMe | Kernel NVMe driver | **DOCA SNAP** | NVMe emulation/virtualization | [DOCA Services](https://docs.nvidia.com/doca/sdk/doca+services/index.html) |
| **Storage** | Compression | zlib / lz4 (CPU) | **DOCA Compress** | Hardware compression engine | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Storage** | DMA | Kernel DMA | **DOCA DMA** | Hardware DMA acceleration | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **GPU** | GPUDirect RDMA | N/A | **DOCA GPUNetIO** | Direct GPU-NIC path, bypass CPU | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Telemetry** | Flow Monitoring | tcpdump / sFlow (CPU) | **DOCA Flow Inspector** | Hardware packet mirroring | [DOCA Services](https://docs.nvidia.com/doca/sdk/doca+services/index.html) |
| **Telemetry** | Metrics Export | Prometheus exporters | **DOCA Telemetry Service** | DPU-native telemetry | [DOCA Services](https://docs.nvidia.com/doca/sdk/doca+services/index.html) |
| **Time Sync** | PTP | linuxptp (`apt install linuxptp`) | **DOCA PTP Service** | Hardware timestamping | [DOCA Services](https://docs.nvidia.com/doca/sdk/doca+services/index.html) |
| **Networking** | Congestion Control | Kernel TCP CC | **DOCA PCC** | Programmable Congestion Control | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Networking** | L2 Forwarding | Linux bridge | **DOCA L2 Forwarding** | Hardware L2 switching | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Networking** | RegEx / DPI | Software regex (CPU) | **DOCA RegEx** | Hardware regex engine | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **Integrity** | File Hashing | sha256sum (CPU) | **DOCA SHA** | Hardware SHA acceleration | [DOCA SDK](https://docs.nvidia.com/doca/sdk/index.html) |
| **SR-IOV** | VF Management | Kernel sysfs / ip link | **DOCA + ASAP²** | VF representors for offload | [Virtual Switch on BlueField](https://docs.nvidia.com/networking/display/bluefielddpuosv385/virtual+switch+on+bluefield+dpu) |

## ASAP² (Accelerated Switching and Packet Processing) Architecture

From [Virtual Switch on BlueField DPU](https://docs.nvidia.com/networking/display/bluefielddpuosv385/virtual+switch+on+bluefield+dpu):

> "ASAP² allows offloading the datapath by programming the NIC embedded switch (eSwitch) and avoiding the need to pass every packet through the Arm cores. The control plane remains the same as working with standard OVS."

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ASAP² OFFLOAD MODEL                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Control Plane (Software - Arm Cores)                                 │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  OVS vswitchd    │    TC flower    │    DOCA Flow API           │  │
│   │  (unchanged)     │    (unchanged)  │    (native)                │  │
│   └────────┬─────────┴────────┬────────┴────────┬────────────────────┘  │
│            │                  │                 │                       │
│            ▼                  ▼                 ▼                       │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │              eSwitch Flow Tables (Hardware)                      │  │
│   │  • VXLAN encap/decap         • VLAN push/pop                    │  │
│   │  • 5-tuple matching          • Header rewrite                   │  │
│   │  • Forward/drop/mirror       • Statistics counters              │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   Data Plane: 400Gbps line rate, zero CPU involvement                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## TC Flower Hardware Offload Examples

From [OVS-Kernel Hardware Offloads](https://docs.nvidia.com/doca/archive/2-7-0/ovs-kernel+hardware+offloads/index.html):

```bash
# Drop rule - hardware only (skip_sw bypasses kernel)
tc filter add dev enp3s0f0_0 ingress protocol ip prio 1 \
    flower skip_sw dst_mac aa:bb:cc:dd:ee:ff \
    action drop

# VXLAN decapsulation rule
tc filter add dev vxlan_sys_4789 ingress protocol ip prio 1 \
    flower enc_dst_ip 10.0.0.1 enc_key_id 100 enc_dst_port 4789 \
    action tunnel_key unset \
    action mirred egress redirect dev enp3s0f0_0

# VLAN push rule
tc filter add dev enp3s0f0_0 ingress protocol 802.1Q prio 1 \
    flower skip_sw vlan_id 100 \
    action vlan push id 200 \
    action mirred egress redirect dev enp3s0f0_1

# Check offloaded flows
tc -s filter show dev enp3s0f0_0 ingress

# Verify OVS offload
ovs-appctl dpctl/dump-flows type=offloaded
```

## OVS-DOCA Configuration Example

```bash
# Enable hardware offload mode
ovs-vsctl set Open_vSwitch . other_config:hw-offload=true

# Create bridge with netdev datapath (required for DOCA)
ovs-vsctl add-br br-int -- set bridge br-int datapath_type=netdev

# Add physical port representor
ovs-vsctl add-port br-int pf0hpf -- \
    set interface pf0hpf type=dpdk \
    options:dpdk-devargs=0000:03:00.0,representor=pf0

# Add VF representor
ovs-vsctl add-port br-int vf0 -- \
    set interface vf0 type=dpdk \
    options:dpdk-devargs=0000:03:00.0,representor=vf0

# Add VXLAN tunnel port
ovs-vsctl add-port br-int vxlan0 -- \
    set interface vxlan0 type=vxlan \
    options:remote_ip=flow \
    options:key=flow

# Verify hardware offload status
ovs-vsctl get Open_vSwitch . other_config
ovs-appctl dpctl/dump-flows type=offloaded
```

---

## NVIDIA DOCA Platform Framework (DPF) - Kubernetes CRDs

When the DPU is managed via Kubernetes, NVIDIA provides the **DOCA Platform Framework (DPF)** with declarative CRDs.

### DPF CRD Reference

| CRD | API Group | Purpose | Documentation |
|-----|-----------|---------|---------------|
| **DPFOperatorConfig** | `operator.dpu.nvidia.com` | Configures DPF operator, hosted cluster type | [DPF Docs](https://docs.nvidia.com/networking/display/dpf2571/doca+platform+framework+v25-7-0) |
| **DPUCluster** | `provisioning.dpu.nvidia.com` | Defines K8s cluster on BlueField DPUs | [DPUCluster](https://docs.nvidia.com/networking/display/dpf2504/dpucluster) |
| **DPUSet** | `provisioning.dpu.nvidia.com` | Manages DPU groups, BFB upgrades, provisioning | [DPUSet](https://docs.nvidia.com/networking/display/dpf2504/dpuset) |
| **DPUService** | `svc.dpu.nvidia.com` | Deploys containerized services to DPUs | [DPUService Dev Guide](https://docs.nvidia.com/networking/display/dpftest1/dpuservice+development+guide) |
| **DPUDeployment** | `svc.dpu.nvidia.com` | Lifecycle management of DPUServices + DPUSets | [DPF Docs](https://docs.nvidia.com/networking/display/dpf2571) |
| **DPUServiceChain** | `svc.dpu.nvidia.com` | Chains multiple DPUServices together | [DPF Docs](https://docs.nvidia.com/networking/display/dpf2571) |
| **DPUServiceIPAM** | `svc.dpu.nvidia.com` | IP address management for DPU services | [DPF Docs](https://docs.nvidia.com/networking/display/dpf2571) |
| **DPUServiceNAD** | `svc.dpu.nvidia.com` | NetworkAttachmentDefinitions for DPU pods | [DPUServiceNAD](https://docs.nvidia.com/networking/display/dpf2571/dpuservicenad) |
| **DPUServiceCredentialRequest** | `svc.dpu.nvidia.com` | Credential management for DPU services | [DPF Docs](https://docs.nvidia.com/networking/display/dpf2571) |

### Example: DPUCluster

```yaml
apiVersion: provisioning.dpu.nvidia.com/v1alpha1
kind: DPUCluster
metadata:
  name: dpu-cluster-production
  namespace: dpf-operator-system
spec:
  type: kamaji                    # Hosted control plane type
  maxUnavailable: 1               # Rolling update strategy
  paused: false
  version: "1.29.0"               # Kubernetes version on DPUs
```

### Example: DPUSet

```yaml
apiVersion: provisioning.dpu.nvidia.com/v1alpha1
kind: DPUSet
metadata:
  name: bluefield3-workers
  namespace: dpf-operator-system
spec:
  dpuSelector:
    matchLabels:
      dpu-type: bluefield-3
  bfb:
    version: "3.2.1"              # DOCA BFB version
  nodeEffect:
    taint:
      key: "dpu.nvidia.com/dedicated"
      value: "true"
      effect: NoSchedule
```

### Example: DPUService

```yaml
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUService
metadata:
  name: ovs-doca
  namespace: dpf-operator-system
spec:
  helmChart:
    source:
      repoURL: https://helm.ngc.nvidia.com/nvidia/doca
      chartName: ovs-doca
      version: "3.2.1"
  serviceConfiguration:
    serviceDaemonSet:
      nodeSelector:
        node-role.kubernetes.io/dpu-worker: ""
    configMapGenerator:
      name: ovs-doca-config
      data:
        hw-offload: "true"
        datapath-type: "netdev"
```

### Example: DPUDeployment

```yaml
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUDeployment
metadata:
  name: ai-factory-networking
  namespace: dpf-operator-system
spec:
  dpus:
    dpuSelector:
      matchLabels:
        cluster: ai-factory
        role: networking
  services:
    - name: ovs-doca
      version: "3.2.1"
    - name: doca-firewall
      version: "3.2.1"
      annotations:
        dpu.nvidia.com/doca-version: ">= 3.0"
    - name: doca-telemetry
      version: "3.2.1"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

### Example: DPUServiceChain

```yaml
apiVersion: svc.dpu.nvidia.com/v1alpha1
kind: DPUServiceChain
metadata:
  name: security-chain
  namespace: dpf-operator-system
spec:
  services:
    - name: doca-firewall
      order: 1
    - name: doca-flow-inspector
      order: 2
    - name: ovs-doca
      order: 3
  defaultAction: forward
```

### DPF Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HOST CONTROL PLANE (x86 K8s Cluster)                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                    DPF Operator                                  │  │
│   │                                                                  │  │
│   │  Watches CRDs:                 Reconciles:                      │  │
│   │  • DPFOperatorConfig           • Provisions DPU clusters        │  │
│   │  • DPUCluster                  • Deploys DPUServices            │  │
│   │  • DPUSet                      • Manages BFB lifecycle          │  │
│   │  • DPUService                  • Handles rolling updates        │  │
│   │  • DPUDeployment                                                │  │
│   │  • DPUServiceChain                                              │  │
│   │                                                                  │  │
│   └──────────────────────────┬──────────────────────────────────────┘  │
│                              │                                          │
│                              │ Manages via Kubernetes API               │
│                              ▼                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                    DPU CLUSTER (ARM K8s on BlueField DPUs)              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐             │
│   │  BlueField 1  │  │  BlueField 2  │  │  BlueField N  │             │
│   │  (DPU Node)   │  │  (DPU Node)   │  │  (DPU Node)   │             │
│   │               │  │               │  │               │             │
│   │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │             │
│   │ │OVS-DOCA   │ │  │ │OVS-DOCA   │ │  │ │OVS-DOCA   │ │             │
│   │ │DaemonSet  │ │  │ │DaemonSet  │ │  │ │DaemonSet  │ │             │
│   │ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │             │
│   │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │             │
│   │ │Firewall   │ │  │ │Firewall   │ │  │ │Firewall   │ │             │
│   │ │DaemonSet  │ │  │ │DaemonSet  │ │  │ │DaemonSet  │ │             │
│   │ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │             │
│   │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │             │
│   │ │Telemetry  │ │  │ │Telemetry  │ │  │ │Telemetry  │ │             │
│   │ │DaemonSet  │ │  │ │DaemonSet  │ │  │ │DaemonSet  │ │             │
│   │ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │             │
│   └───────────────┘  └───────────────┘  └───────────────┘             │
│                                                                         │
│   All services deployed declaratively via DPUService CRDs              │
│   GitOps compatible (Flux, ArgoCD)                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## DPU Management Model Comparison

| Approach | Tools | Scaling | GitOps | Best For |
|----------|-------|---------|--------|----------|
| **Direct SSH + Ansible** | ssh, ansible-playbook | Ansible inventory | Playbooks in git | Slurm clusters, simple setups |
| **Containers (no K8s)** | podman, docker-compose | Ansible-managed | Compose files in git | Slurm clusters, DOCA services |
| **K8s on DPU only** | K3s, MicroK8s per DPU | Manual or Ansible | Flux/Argo per DPU | Isolated DPU management |
| **DPF (full K8s)** | DPF Operator + CRDs | Native K8s scaling | Full GitOps with CRDs | K8s inference clusters |

---

## Key Documentation Links

| Resource | URL |
|----------|-----|
| **DOCA SDK Documentation (v3.2.1 LTS)** | [docs.nvidia.com/doca/sdk](https://docs.nvidia.com/doca/sdk/index.html) |
| **DOCA Platform Framework (DPF) v25.7.0** | [docs.nvidia.com/networking/display/dpf2571](https://docs.nvidia.com/networking/display/dpf2571/doca+platform+framework+v25-7-0) |
| **DPF GitHub Repository** | [github.com/NVIDIA/doca-platform](https://github.com/NVIDIA/doca-platform) |
| **DOCA Services** | [docs.nvidia.com/doca/sdk/doca+services](https://docs.nvidia.com/doca/sdk/doca+services/index.html) |
| **DOCA Developer Portal** | [developer.nvidia.com/networking/doca](https://developer.nvidia.com/networking/doca) |
| **DPU-DOCA Documentation Hub** | [docs.nvidia.com/networking/dpu-doca](https://docs.nvidia.com/networking/dpu-doca/index.html) |
| **OVS in DOCA Guide** | [docs.nvidia.com/doca/sdk/openvswitch-acceleration](https://docs.nvidia.com/doca/sdk/openvswitch-acceleration---ovs-in-doca/index.html) |
| **Virtual Switch on BlueField** | [docs.nvidia.com/networking/display/bluefielddpuosv385](https://docs.nvidia.com/networking/display/bluefielddpuosv385/virtual+switch+on+bluefield+dpu) |
