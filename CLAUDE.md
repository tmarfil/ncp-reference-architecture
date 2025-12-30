***Let's discuss one concept at a time. Create a todo.md of all concepts to cover based on the list the logical isolation, traffic segregation, and resiliency techniques defined across the sources for **NVIDIA Cloud Partner (NCP)** reference architectures. Sort the list for best logical learning order and fill in any missing techniques Once I confirm I'm clear on a concept, summarize our discussion in a dedicated GitHub flavored markdown file for the topic, mark the topic completed in the todo.md, and move on to the next one. The focus is on conceptual understanding and not specific details. Include brief, illustrative config examples for NVIDIA / Cumulus Linux only when they help clarify a concept.***

I am a Networking, Linux, NVIDIA AI Factory, and Kubernetes expert. I'm design a reference architecture for an NVIDIA AI Factory.

There are two networks:

- dedicated east-west fabric for GPU-to-GPU communication (WE WILL NOT BE DISCUSSING THIS AT ALL)

- converged north-south traffic for everything else: (FOCUS OF OUR SESSION)
    - Ingress / Egress to cluster (inference)
    - In band management
    - Kubernetes Networking (if running K8s)
    - High Speed Storage (some solutions depend on RoCEv2 others on simple TCP/IP)
    
The Reference Design calls for a 'converged' or shared ethernet spine-leaf design for north-south based on NVIDIA Spectrum-X switches. Each switch as 256 x 400G ports.
Each compute node has 2 x Bluefield-4 DPUs with 4 x 400G ports. Each compute node has 4 x GPUs.

### Logical Isolation and Multitenancy
*   **VXLAN (Virtual Extensible LAN)**: This is the primary technique used to apply tenancy and ensure isolation across the **Converged Ethernet Fabric**.
*   **BGP EVPN (Border Gateway Protocol Ethernet VPN)**: This serves as the standards-based control plane used for state synchronization and managing the various **VXLAN** segments.
*   **VRF (Virtual Routing and Forwarding)**: Utilized in **distributed gateway designs**, this ensures that every tenant mapped to the infrastructure has its own independent subnet gateway for flexible network configuration.
*   **VLAN (Virtual Local Area Network) Tagging**: This is the preferred method for isolation within **HPS (High-Performance Storage)** environments because it simplifies fabric scalability and management compared to VRF-based storage isolation.
*   **Partition Keys (PKey)**: When **InfiniBand** is used for the compute network, **VXLAN ID** information from the converged network is mapped to a **PKey** to maintain consistent isolation across both fabrics.

### Traffic Quality and Management
*   **RoCEv2 (RDMA over Converged Ethernet version 2)**: This protocol is supported for high-performance data movement while maintaining multitenant traffic isolation.
*   **Adaptive Routing**: A hardware capability of the **Spectrum-6** switch that optimizes traffic flow by dynamically selecting the best path to avoid congestion.
*   **Hardware-Based Congestion Control**: Native support within the switching fabric ensures that high-bandwidth storage or management traffic does not interfere with other tenants in a shared environment.

### Network Resiliency and Host Connectivity
*   **EVPN-MH (Ethernet VPN Multihoming)**: This is the recommended standards-based solution to eliminate single points of failure by connecting a host to two separate leaf switches for active-active forwarding and seamless failover.
*   **Hardware-Accelerated ECMP (Equal-Cost Multi-Path)**: When using **BlueField DPUs (Data Processing Units)** in DPU Mode, the DPU acts as a Layer-3 router, performing per-flow load balancing and failover entirely in hardware, which eliminates the need for host-side bonding.
*   **VF-LAG (Virtual Function Link Aggregation)**: A capability used to hardware-accelerate bonded network interfaces through the **OVS-DOCA (Open vSwitch Data Center Infrastructure-on-a-Chip Architecture)** virtual switch.
*   **MC-LAG (Multi-Chassis Link Aggregation)**: A proprietary method used to make two physical switches appear as a single logical entity to a host; however, the sources generally **do not recommend** this due to its complexity and lower port efficiency compared to **EVPN-MH**.

### Security and Infrastructure Control
*   **Zero-Trust Tenant Isolation**: Anchored by the **BlueField Advanced Secure Trusted Resource Architecture**, this technique shifts security enforcement from the network fabric directly into the **DPU** hardware, closer to the workload.
*   **NVIDIA DOCA Microservices**: Containerized services that provide a unified framework to secure and scale network, storage, and security operations.
*   **Physical Isolation**: This is strictly maintained for the **OOB (Out-of-Band)** management network, ensuring that hardware management ports are physically separated from the converged fabric used for tenant data and storage traffic.
