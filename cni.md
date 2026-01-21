# CNI Types and Characteristics

## CNIs

Kubernetes **Container Network Interface (CNI)** plugins are categorized primarily by their data plane technology (e.g., eBPF vs. iptables) and their network architecture (overlay vs. underlay). Choosing a CNI is a strategic decision impacting cluster performance, security policy enforcement, and observability

### Popular CNI Types & Characteristics

| CNI Plugin | Primary Technology | Key Characteristics | Best Use Case |
|-----------|-------------------|---------------------|---------------|
| Cilium | eBPF | High performance, deep L7 observability (Hubble), and identity-based security policies. | High-traffic, latency-sensitive production environments. |
| Calico | BGP / iptables | Robust L3 routing, industry-standard network policy engine, and high scalability (10k+ nodes). | Enterprise-grade security and large-scale hybrid/multi-cloud deployments. |
| Flannel | VXLAN | Minimalist, very easy to set up, but lacks network policy support and has overlay overhead. | Development, testing, or small clusters where simplicity is prioritized over security. |
| Canal | Hybrid | Combines Flannel's simple networking with Calico's advanced network policy enforcement. | Teams needing a middle ground between simplicity and security policy. |
| Weave Net | Mesh Overlay | Built-in encryption (IPsec) and an automatic peer-to-peer mesh that is highly resilient. | Smaller clusters requiring simple out-of-the-box traffic encryption. |

### Specialized & Cloud-Native CNIs

- **Cloud-Native CNIs (AWS VPC, Azure CNI, GKE CNI)**: Assign routable IPs directly from the cloud provider's virtual network. They offer high performance and tight integration with cloud firewalls but are less portable across different providers.  
- **Antrea**: Optimized for VMware environments and private clouds, utilizing Open vSwitch for high-performance networking and security.
- **Submariner**: Specialized for connecting pods and services across multiple clusters in hybrid or multi-cloud setups. 

### Core Network Models

CNI plugins generally implement one of two networking models: 
- **Encapsulated (Overlay)**: Uses technologies like VXLAN or Geneve to create a virtual Layer 2 network over an existing Layer 3 topology. This is easier to set up but introduces packet overhead.
- **Unencapsulated (Underlay/Direct Routing)**: Uses protocols like BGP to route packets directly between nodes. This provides superior performance and lower latency but requires more complex integration with physical network hardware. 

### Selection Criteria

- **Security**: If you require zero-trust microsegmentation, **Cilium** and **Calico** are the primary choices due to their advanced policy engines.
- **Performance**: eBPF-based plugins like **Cilium** generally outperform traditional iptables-based solutions by bypassing parts of the Linux kernel network stack.
- **Scalability**: **Calico** is widely recognized for handling massive clusters with thousands of nodes and hundreds of thousands of pods.
- **Kernel Support**: eBPF-powered CNIs require modern Linux kernels (typically 5.2+), which may be a constraint in older enterprise environments

## eBPF

**eBPF (Extended Berkeley Packet Filter)** has revolutionized network engineering by allowing custom programs to run directly within a sandboxed environment in the Linux kernel without requiring reboots or kernel modifications.  

### What is eBPF?

**eBPF** is often described as "superpowers for the kernel". It is a versatile technology that intercepts events—such as network packets, system calls, or function entries—and executes safe, high-performance programs in response. 
- **Safety**: A "verifier" checks every program before execution to ensure it won't crash the system or loop infinitely.
- **Efficiency**: Programs are JIT-compiled into native machine code for near-hardware execution speeds.
- **Hook-based:** It can attach to various points like XDP (earliest packet arrival) or TC (traffic control) to drop, redirect, or modify data.

### eBPF vs. iptables Comparison

The shift from iptables to eBPF is driven by the need for massive scalability in modern Kubernetes environments where thousands of services create unwieldy rule chains

| Feature | iptables (Traditional) | eBPF (Modern) |
|---------|------------------------|---------------|
| Search Mechanism | Linear Traversal: Checks every rule sequentially until a match is found. | Hash Table Lookups: Direct, O(1) constant-time searches regardless of rule count. |
| Performance | Performance degrades significantly as the number of rules/services grows. | High, consistent performance even with hundreds of thousands of rules. |
| Latency | Higher latency due to packet movement through long rule chains. | Ultra-low latency by processing packets at the earliest possible kernel hook. |
| Updates | Requires reloading the entire rule set for any single change, causing "blips". | Atomic Updates: Instantly updates specific entries in kernel memory (Maps) with no downtime. |
| Observability | Limited; requires separate agents or sidecars to see what's happening. | Deep, built-in visibility (L3-L7) without extra overhead (e.g., Hubble). |
| Security Scope | Primarily IP/port-based (L3/L4) firewall rules. | Identity-based and API-aware security (L3-L7) enforced in the kernel. |
| Compatibility | Works on almost all Linux kernels, even very old ones. | Requires modern kernels (typically 5.2+) for full functionality. |

### Kubernetes CNI Technology Comparison

Kubernetes CNI (Container Network Interface) plugins are primarily classified by their **data plane technology**. While many modern plugins are transitioning to eBPF for performance and observability, several established plugins still rely on `iptables`, `IPVS`, or specialized userspace/hardware technologies

| CNI Plugin | Primary Data Plane | Secondary/Alternative | Key Characteristics in 2026 |
|-----------|-------------------|---------------------|------------------------------|
| Cilium | eBPF | — | Native eBPF from the start; provides deep L7 visibility (Hubble) and identity-based security. |
| Calico | iptables | eBPF, IPVS, VPP | Highly versatile; uses BGP for routing and supports an advanced eBPF data plane for performance. |
| Antrea | Open vSwitch (OVS) | — | Optimized for VMware environments; uses OVS for high-performance L2/L3 networking and security. |
| Flannel | VXLAN | host-gw, WireGuard | The simplest CNI; uses basic Linux bridging and VXLAN overlays with no network policy support. |
| Kube-router | IPVS | iptables | Uses IPVS for service load balancing and BGP for pod networking; lean and performance-focused. |
| Canal | VXLAN (Flannel) | iptables (Calico) | A hybrid that uses Flannel for the data path and Calico's iptables rules for network policies. |
| OVN-Kubernetes | OVS / OVN | — | Enterprise SDN that can bypass the CPU for established flows; common in Red Hat OpenShift. |
| Weave Net | VXLAN | — | Uses a mesh overlay with gossip-based discovery; best for smaller clusters requiring easy encryption. |
| AWS/Azure/GKE CNI | VPC/VNet Native | — | Uses the cloud provider's native networking (e.g., VPC ENIs) for direct, routable IP addresses. |
| Userspace CNI | DPDK / VPP | — | Designed for specialized high-throughput/low-latency needs (Telcos); runs networking in userspace. |  

**Technology Summaries**  

- **eBPF-Based**: The modern standard for high-scale environments. It replaces slow iptables chains with constant-time hash table lookups, significantly reducing CPU overhead and latency.
- **iptables-Based**: The traditional "legacy" method. While widely compatible, it suffers from performance degradation as the number of services and rules grows due to its linear search mechanism.
- **OVS (Open vSwitch)**: A programmable software switch that provides advanced SDN features. It is often used in enterprise environments that require fine-grained control over packet flows.
- **IPVS (IP Virtual Server)**: A transport-layer load balancing solution built into the Linux kernel. It is faster than iptables for service routing but lacks the security policy flexibility of eBPF. 

