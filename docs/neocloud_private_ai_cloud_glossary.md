# NeoCloud / Private AI Cloud Component Glossary

This glossary covers the core hardware, networking, storage, operating-system, orchestration, control-plane, and operations components commonly found in NeoCloud and private AI cloud architectures.

| Short Name | Full Name | Short Description |
|---|---|---|
| GPU | Graphics Processing Unit | Primary accelerator used for AI training and inference. Optimized for highly parallel matrix and tensor operations. |
| CPU | Central Processing Unit | General-purpose processor that runs the operating system, orchestration agents, data preparation, networking control, and application support logic. |
| HBM | High Bandwidth Memory | Very high-speed memory attached directly to a GPU. Stores model weights, activations, KV cache, and intermediate tensors during AI execution. |
| RAM | Random Access Memory | Main system memory used by CPUs for operating-system processes, applications, data loading, caching, and preprocessing. |
| NVLink | NVIDIA NVLink | High-bandwidth, low-latency interconnect used for direct GPU-to-GPU communication within a scale-up domain. |
| NVSwitch | NVIDIA NVSwitch | Switching fabric that connects multiple GPUs over NVLink so they can communicate at very high bandwidth. |
| NVLink Domain | NVLink Scale-Up Domain | Group of GPUs connected by NVLink/NVSwitch and able to communicate much faster than GPUs connected only through the scale-out network. |
| PCIe | Peripheral Component Interconnect Express | General-purpose high-speed bus connecting CPUs with GPUs, NICs, NVMe drives, and other devices inside a server. |
| PCIe Root | PCI Express Root Complex | CPU-side controller for a PCIe hierarchy. GPU-to-NIC and GPU-to-storage locality relative to the PCIe root can affect performance. |
| NUMA | Non-Uniform Memory Access | Server architecture where CPUs access local memory faster than memory attached to another CPU socket. Important for CPU, GPU, and NIC locality. |
| NIC | Network Interface Card | Network adapter connecting a server to compute, storage, tenant, or management networks. |
| ConnectX | NVIDIA ConnectX | High-performance NVIDIA network adapter family commonly used for InfiniBand and high-speed Ethernet/RoCE GPU clusters. |
| DPU | Data Processing Unit | Programmable infrastructure processor that can offload networking, security, storage, and virtualization tasks from host CPUs. |
| NVMe | Non-Volatile Memory Express | High-speed local SSD storage commonly used for caching datasets, model files, checkpoints, container layers, and temporary data. |
| SSD | Solid-State Drive | Persistent flash storage used locally in servers or as part of shared storage systems. |
| BMC | Baseboard Management Controller | Out-of-band controller used to power-cycle, inspect, configure, and recover a physical server independently of the host operating system. |
| BIOS/UEFI | Basic Input/Output System / Unified Extensible Firmware Interface | Low-level server firmware that initializes hardware and exposes configuration settings before the operating system starts. |
| Firmware | Device Firmware | Embedded software running on GPUs, NICs, BMCs, switches, SSDs, and other hardware components. Version consistency is important for reliability and performance. |
| ECC | Error Correcting Code | Memory protection mechanism that detects and corrects certain bit errors. GPU ECC metrics are important indicators of hardware health. |
| PSU | Power Supply Unit | Converts incoming electrical power into the voltages required by server components. |
| PDU | Power Distribution Unit | Distributes electrical power within a rack and provides monitoring and sometimes remote switching or power control. |
| UPS | Uninterruptible Power Supply | Provides temporary battery-backed power and protection against power interruptions or instability. |
| CDU | Coolant Distribution Unit | Controls and circulates coolant between the data-center water system and liquid-cooled racks or servers. |
| DLC | Direct Liquid Cooling | Cooling method where liquid removes heat directly from CPUs, GPUs, or other hot components, usually through cold plates. |
| Cold Plate | Liquid Cooling Cold Plate | Heat-transfer plate mounted directly on a high-power component such as a GPU or CPU. |
| Rack | Equipment Rack | Physical cabinet containing GPU servers or compute trays, networking, power distribution, and sometimes cooling infrastructure. |
| Compute Tray | GPU Compute Tray | Rack-mounted compute module containing CPUs, GPUs, memory, NICs, and related components. Common in rack-scale AI systems. |
| NVL72 | NVIDIA GB200/GB300 NVL72 | Rack-scale NVIDIA architecture connecting 72 Blackwell GPUs in one large NVLink domain. |
| Scale-Up | Scale-Up Architecture | Increases tightly coupled compute capacity within a single system or NVLink domain. NVLink/NVSwitch are key scale-up technologies. |
| Scale-Out | Scale-Out Architecture | Increases total compute by connecting many servers or racks through a high-speed network such as InfiniBand or Ethernet/RoCE. |
| IB | InfiniBand | High-performance network fabric widely used in HPC and AI clusters for low-latency, high-bandwidth RDMA communication. |
| Ethernet | Ethernet | Standard packet-networking technology widely used for tenant traffic, storage, management, and increasingly high-performance AI fabrics. |
| RoCE | RDMA over Converged Ethernet | Technology that provides RDMA semantics over Ethernet and is commonly used for high-performance AI scale-out networking. |
| RDMA | Remote Direct Memory Access | Allows data to move directly between memory regions on different systems with minimal CPU involvement. |
| GDR | GPUDirect RDMA | NVIDIA technology allowing GPU memory to communicate directly across RDMA-capable networks without routing the data through CPU system memory. |
| GPUDirect Storage | NVIDIA GPUDirect Storage | Allows storage data to move more directly between storage devices and GPU memory, reducing CPU and system-memory involvement. |
| NCCL | NVIDIA Collective Communications Library | Library used by distributed AI frameworks for efficient GPU communication such as AllReduce, AllGather, ReduceScatter, and Broadcast. |
| Leaf Switch | Leaf Network Switch | Switch that connects directly to GPU servers or racks in a leaf-spine topology. |
| Spine Switch | Spine Network Switch | High-capacity switch that connects leaf switches together and carries traffic across the cluster. |
| Core Switch | Core Network Switch | Additional higher-level switching tier sometimes used to connect large network pods or scalable units in very large clusters. |
| Leaf-Spine | Leaf-Spine Network Topology | Network design where servers connect to leaf switches and leaves connect to multiple spines, providing predictable east-west paths and redundancy. |
| Rail | Network Rail | Parallel network path associated with specific GPU/NIC groups. Rail-optimized designs preserve locality and reduce contention in large GPU fabrics. |
| Bisection BW | Bisection Bandwidth | Total bandwidth available between two halves of a network. Important for determining whether large distributed jobs can communicate without central bottlenecks. |
| Oversubscription | Network Oversubscription Ratio | Ratio between downstream endpoint bandwidth and available uplink bandwidth. Lower oversubscription is generally preferred for tightly coupled AI training. |
| East-West | East-West Traffic | Traffic moving inside the cloud or cluster, especially GPU-to-GPU, service-to-service, and server-to-storage communication. |
| North-South | North-South Traffic | Traffic entering or leaving the cloud, such as API requests, SSH, user access, inference requests, or Internet traffic. |
| Compute Fabric | GPU Scale-Out Compute Fabric | High-performance network connecting GPU servers or racks for distributed training and other tightly coupled workloads. |
| Storage Fabric | Storage Network Fabric | Network carrying data between compute nodes and shared storage systems. May be physically separate from the GPU compute fabric. |
| Tenant Network | Tenant / Service Network | Customer-facing network used for VMs, containers, APIs, inference endpoints, SSH, and application traffic. |
| OOB Network | Out-of-Band Management Network | Separate management network used to reach BMCs, switches, and infrastructure even when production networks or host operating systems fail. |
| Shared FS | Shared File System | High-performance filesystem accessible by many GPU nodes simultaneously for datasets, models, checkpoints, and shared files. |
| Lustre | Lustre Parallel File System | Widely used distributed parallel filesystem for HPC and AI workloads requiring very high aggregate throughput. |
| BeeGFS | BeeGFS Parallel File System | Distributed parallel filesystem designed for high-performance compute and AI workloads. |
| WEKA | WEKA Data Platform | Commercial high-performance distributed storage platform frequently used for AI and GPU clusters. |
| VAST | VAST Data Platform | Commercial scale-out data platform used for high-throughput AI, analytics, and unstructured data workloads. |
| Object Store | Object Storage | Durable, massively scalable storage such as S3-compatible storage, GCS, or Azure Blob, commonly used for datasets, models, logs, and long-term checkpoints. |
| Checkpoint | Training Checkpoint | Persisted snapshot of model and optimizer state that allows long-running training jobs to resume after failure or interruption. |
| Cache | Data / Model Cache | Faster storage layer used to keep frequently accessed datasets, model weights, or artifacts closer to GPUs. |
| PXE | Preboot Execution Environment | Network-based boot mechanism commonly used to automate bare-metal operating-system installation and recovery. |
| Linux | Linux Operating System | Primary host operating system used on most GPU compute nodes and cloud infrastructure servers. |
| NVIDIA Driver | NVIDIA GPU Driver | Host software allowing Linux and user-space software to communicate with NVIDIA GPUs. |
| CUDA | Compute Unified Device Architecture | NVIDIA's programming platform, runtime, compiler, and libraries for GPU-accelerated computing. |
| PyTorch | PyTorch | Popular AI framework that uses CUDA and related libraries to execute training and inference workloads on NVIDIA GPUs. |
| Container | Application Container | Isolated userspace environment packaging application code and dependencies while sharing the host operating-system kernel. |
| OCI | Open Container Initiative | Standards defining common container image and runtime formats used by Docker, containerd, Kubernetes, and related systems. |
| containerd | containerd | Common container runtime used by Kubernetes to pull images and create and manage containers. |
| K8s | Kubernetes | Container orchestration platform used to schedule, run, scale, restart, and manage distributed services and increasingly AI workloads. |
| Pod | Kubernetes Pod | Basic Kubernetes execution unit containing one or more containers scheduled together on a node. |
| Kubelet | Kubernetes Kubelet | Node-side Kubernetes agent that manages pods and reports node resources and health to the control plane. |
| Device Plugin | NVIDIA Kubernetes Device Plugin | Kubernetes integration that discovers NVIDIA GPUs on a node and exposes them as schedulable resources. |
| Slurm | Simple Linux Utility for Resource Management | HPC workload manager and scheduler widely used for batch computing and large distributed AI training jobs. |
| slurmd | Slurm Node Daemon | Agent running on each Slurm compute node that launches tasks and reports node state to the Slurm controller. |
| Scheduler | Resource Scheduler | Software that decides which workloads receive which GPUs, CPUs, memory, nodes, racks, and other resources. |
| Gang Scheduling | Gang Scheduling | Scheduling method where all resources required by a distributed job are allocated together before the job starts. |
| Queue | Workload Queue | Ordered set of jobs waiting for compute resources, often governed by priority, quotas, reservations, or fair-share policies. |
| Quota | Resource Quota | Policy defining how much compute, storage, or other capacity a tenant, project, or user may consume. |
| Reservation | Resource Reservation | Capacity held for a specific customer, project, time period, or workload rather than being offered to the general pool. |
| Preemption | Workload Preemption | Stopping or pausing lower-priority workloads so higher-priority workloads can use the resources. |
| Spot | Spot / Preemptible Capacity | Discounted capacity that can be reclaimed by the provider when higher-priority demand appears. |
| Fragmentation | Resource Fragmentation | Condition where total free GPUs exist but are scattered across servers or topology domains and cannot satisfy a tightly coupled request. |
| Topology-Aware Scheduling | Topology-Aware Scheduling | Scheduling that considers NVLink domains, PCIe locality, NIC placement, rack placement, and network topology rather than only free GPU count. |
| Data Locality | Data Locality | Placement principle that schedules workloads close to required datasets or caches to reduce storage and network movement. |
| Affinity | Scheduling Affinity | Rule or preference that places workloads near selected resources, workloads, racks, zones, or topology domains. |
| Anti-Affinity | Scheduling Anti-Affinity | Rule that deliberately separates workloads across servers, racks, or failure domains to improve resiliency. |
| Control Plane | Control Plane | Software layer that makes infrastructure decisions such as scheduling, authorization, provisioning, policy, placement, and desired state. |
| Data Plane | Data Plane | Infrastructure that actually executes workloads and moves data, including GPUs, containers, networks, and storage paths. |
| Desired State | Desired-State Configuration | Declarative description of how infrastructure or workloads should be configured; management software continuously reconciles actual state toward it. |
| Fleet Manager | Fleet Management System | Software that provisions, configures, upgrades, tests, drains, repairs, and tracks large numbers of physical servers. |
| Provisioner | Bare-Metal Provisioning System | Software that installs operating systems, configures firmware, assigns networking, and prepares new physical nodes for production. |
| Node | Compute Node | Individual physical or virtual machine participating in a Kubernetes, Slurm, or other compute cluster. |
| Cluster | Compute Cluster | Group of connected compute nodes managed together and typically sharing a scheduling, networking, and storage environment. |
| Pod / SU | Network Pod / Scalable Unit | Repeatable group of racks, compute, networking, and sometimes storage used as a building block for very large clusters. |
| Failure Domain | Infrastructure Failure Domain | Set of resources likely to fail together because they share a component such as a server, rack, switch, power feed, cooling loop, or data center. |
| Region | Cloud Region | Geographic cloud location containing one or more data centers or availability zones. |
| AZ | Availability Zone | Isolated infrastructure zone intended to reduce correlated failures across power, networking, and physical facilities. |
| Drain | Node Draining | Process of preventing new workloads from being scheduled on a node while existing workloads finish or migrate before maintenance. |
| Quarantine | Node Quarantine | Operational state in which a suspect or failed server/GPU is removed from schedulable production capacity until it is repaired and retested. |
| Burn-In | Hardware Burn-In Testing | Sustained stress testing of new or repaired hardware to reveal intermittent GPU, memory, network, thermal, or power problems before production use. |
| Qualification | Cluster / Node Qualification | Validation that compute, NVLink, network, storage, thermals, and software meet expected performance and reliability standards. |
| Telemetry | Infrastructure Telemetry | Metrics, logs, events, and traces describing GPU, CPU, network, storage, power, cooling, scheduler, and workload behavior. |
| DCGM | NVIDIA Data Center GPU Manager | NVIDIA tooling and APIs used to monitor GPU health, utilization, memory, errors, thermals, power, and diagnostics. |
| Redfish | DMTF Redfish | Standard REST-based API for managing physical server hardware, BMCs, power, firmware, inventory, and health. |
| IPMI | Intelligent Platform Management Interface | Older widely used standard for remote hardware management through BMCs. |
| Prometheus | Prometheus Monitoring System | Widely used metrics collection and querying system in Kubernetes and cloud-native infrastructure. |
| OTel | OpenTelemetry | Open standard for collecting telemetry such as metrics, logs, and traces from applications and infrastructure. |
| IAM | Identity and Access Management | Authentication and authorization system controlling which users and services may access cloud resources. |
| RBAC | Role-Based Access Control | Authorization model where permissions are assigned to roles and users or services receive those roles. |
| Multi-Tenancy | Multi-Tenant Architecture | Architecture allowing multiple customers or teams to share infrastructure while maintaining isolation, quotas, security, and accounting. |
| VPC | Virtual Private Cloud | Logically isolated customer network environment with private addressing, routing, firewalling, and connectivity controls. |
| VLAN | Virtual LAN | Layer-2 network segmentation mechanism commonly used to separate traffic classes or tenants. |
| VXLAN | Virtual Extensible LAN | Overlay networking technology used to create large-scale virtual Layer-2 networks over an IP fabric. |
| CNI | Container Network Interface | Standard used by Kubernetes networking plugins to configure pod networking and network policy. |
| CSI | Container Storage Interface | Standard used by Kubernetes to provision and attach storage volumes to workloads. |
| LB | Load Balancer | Distributes incoming application or inference requests across multiple healthy service instances. |
| Autoscaler | Workload / Cluster Autoscaler | Software that changes replica counts or cluster capacity based on demand, utilization, or policy. |
| Metering | Resource Metering | Measurement of GPU time, CPU, memory, storage, network, and other resources consumed by customers or workloads. |
| Billing | Usage Billing System | Converts measured resource consumption, reservations, pricing rules, and contracts into customer charges. |
| Portal | Cloud Management Portal | Web interface used by customers and operators to request, view, manage, and monitor cloud resources. |
| CLI | Command-Line Interface | Command-line tool used by developers and operators to interact with cloud APIs and manage resources. |
| API | Application Programming Interface | Programmatic interface used to provision compute, storage, networking, workloads, and other NeoCloud resources. |
| API Gateway | API Gateway | Front-door service that authenticates, routes, rate-limits, and governs cloud API requests. |
| Service Discovery | Service Discovery | Mechanism allowing applications and infrastructure services to locate each other dynamically. |
| DNS | Domain Name System | Maps service and host names to network addresses and is commonly used for application and control-plane service discovery. |
| Secret Store | Secrets Management System | Secure system for storing credentials, keys, tokens, certificates, and other sensitive configuration. |
| KMS | Key Management Service | System for creating, storing, rotating, and controlling cryptographic keys used by infrastructure and tenant workloads. |
| Observability | Infrastructure Observability | Ability to understand infrastructure and workload health through correlated telemetry, topology, events, and performance data. |
| SLO | Service Level Objective | Target reliability or performance goal, such as GPU availability, job start time, storage throughput, or inference latency. |
| SLA | Service Level Agreement | Contractual commitment between provider and customer covering service availability, capacity, performance, support, or other guarantees. |

## Core Mental Model

A NeoCloud or private AI cloud can be viewed as a layered system:

```text
Developer / Customer
        ↓
Portal / API / CLI
        ↓
IAM / Quotas / Billing / Control Plane
        ↓
Kubernetes / Slurm / Scheduler
        ↓
Linux / Containers / CUDA / NCCL
        ↓
GPU / HBM / CPU / RAM / NVMe
        ↓
NVLink / PCIe / NIC
        ↓
InfiniBand or Ethernet/RoCE
        ↓
Shared Storage / Object Storage
        ↓
Racks / Power / Cooling / BMC
```

The key architectural principle is that GPU performance depends on the entire pipeline—not just the GPU itself.

---

# Performance, Efficiency & Observability Vocabulary

The tables above cover the hardware and infrastructure. The tables below cover the **performance,
efficiency, and health-telemetry** layer — the language an observability product actually speaks in, and the
terms that separate "the GPU looks busy" from "the GPU is doing useful work." (For a processor-architecture
reader these map onto familiar ideas: occupancy, roofline, memory- vs compute-bound, throttling, P-states.)

## Compute Efficiency & Performance Metrics

| Short Name | Full Name | Short Description |
|---|---|---|
| FLOP | Floating-Point Operation | A single floating-point op (e.g., a multiply or add). The unit of compute *work*. |
| FLOP/s (FLOPS) | Floating-Point Operations per Second | Compute *rate*. Note the common conflation: "FLOPs" often means a count of operations, "FLOP/s" a rate — context decides. |
| TFLOPS | Tera-FLOP/s | 10¹² FLOP/s. How a GPU's peak compute is rated — and it is **precision-dependent** (FP8 peak ≫ FP32 peak). |
| Peak FLOP/s | Theoretical Peak Compute | The maximum FLOP/s a GPU can sustain at a given precision. The denominator for efficiency metrics. |
| GPU_UTIL | GPU Utilization | The `nvidia-smi` / DCGM metric = fraction of time ≥1 kernel was resident. Measures *presence, not efficiency* — can read ~100% while the GPU is stalled (the "utilization illusion"). |
| SM | Streaming Multiprocessor | The GPU's compute core; schedules and executes warps. (An H100 has ~132 SMs.) |
| Warp | Warp | A group of 32 threads scheduled and executed together on an SM (SIMT). |
| Kernel | GPU Kernel | A function launched to run on the GPU across many threads/blocks. |
| Occupancy | Achieved Occupancy | Active warps ÷ maximum warps per SM. How full the SM's scheduler slots are (bounded by registers/shared memory); higher occupancy helps hide memory latency. |
| SM Active | `SM_ACTIVE` (DCGM) | Fraction of time an SM had ≥1 active warp, averaged over SMs. "Were the cores doing *anything*?" |
| SM Occupancy | `SM_OCCUPANCY` (DCGM) | Resident warps ÷ max, averaged over time and SMs. "Were the cores *full*?" |
| Tensor Core | Tensor Core | Dedicated matrix-multiply-accumulate unit; source of most AI FLOP/s. |
| Tensor Active | `PIPE_TENSOR_ACTIVE` (DCGM) | Fraction of time the tensor pipes were active. The cheapest provider-side proxy for "real AI work." |
| MFU | Model FLOPs Utilization | Achieved *model* FLOP/s ÷ peak FLOP/s. The honest efficiency number; well-tuned large-model training often lands ~35–55%. |
| HFU | Hardware FLOPs Utilization | Like MFU but counts *all* executed FLOPs, including redundant ones (e.g., activation recompute). Always MFU ≤ HFU. |
| Arithmetic Intensity | Arithmetic Intensity | FLOPs performed per byte of memory traffic. Determines whether a kernel is compute- or memory-bound. |
| Roofline | Roofline Model | Plots achievable FLOP/s against arithmetic intensity to show whether a workload is compute-bound or memory-bound. |
| Compute-Bound | Compute-Bound | Limited by math throughput (tensor/ALU), not memory. Typical of large training matmuls. |
| Memory-Bound | Memory-Bound | Limited by HBM bandwidth, not math. Typical of LLM inference decode. |
| MBU | Memory Bandwidth Utilization | Achieved HBM bandwidth ÷ peak. The key efficiency number for memory-bound (inference) workloads. |
| Throughput | Throughput | Useful work rate — tokens/s (LLM), samples/s (training), images/s. |
| Goodput | Goodput | Throughput that actually *counts* — excludes wasted or rolled-back work (failed steps, retries) and requests that miss their SLO. |
| Straggler | Straggler | The slowest worker in a synchronous distributed job; it sets the pace of the whole job. |
| MIG | Multi-Instance GPU | Partitions one physical GPU into isolated instances, each with dedicated SMs and memory slices. |
| MPS | Multi-Process Service | Lets multiple processes share a single GPU's SMs concurrently. |

## Numeric Precision (peak FLOP/s depends on the format)

| Short Name | Full Name | Short Description |
|---|---|---|
| FP32 | 32-bit Floating Point | Full-precision baseline; lowest tensor throughput. |
| TF32 | TensorFloat-32 | NVIDIA tensor-core path with FP32 range but reduced (19-bit) mantissa; near-FP32 accuracy, much faster. |
| FP16 / BF16 | Half / Brain Float 16 | 16-bit formats for training/inference. BF16 trades mantissa for a wider exponent (more stable range). |
| FP8 | 8-bit Floating Point (E4M3 / E5M2) | 8-bit formats; very high throughput on Hopper/Blackwell, common for modern training and inference. |
| INT8 | 8-bit Integer | Integer format widely used for quantized inference. |
| Mixed Precision | Mixed-Precision Training | Compute in low precision while keeping master weights / accumulations in higher precision for stability. |
| Sparsity | 2:4 Structured Sparsity | Hardware skips zeros in a 2-of-4 pattern, giving up to ~2× nominal tensor throughput. |

## GPU Health & Reliability Telemetry

| Short Name | Full Name | Short Description |
|---|---|---|
| Xid | NVIDIA Xid Error | Driver error code emitted to the kernel log (`dmesg`); each number is a fault class (e.g., **Xid 79** = GPU fell off the bus; **Xid 48** = double-bit ECC). |
| SBE / DBE | Single- / Double-Bit ECC Error | SBE is auto-corrected; **DBE is uncorrectable** and usually fatal to the job/host. |
| Row Remapping | HBM Row Remapping | Defect management that retires a bad memory row and maps in a spare. Spares are finite — **exhaustion signals end of life**. |
| SDC | Silent Data Corruption | Wrong compute results with **no error raised** — bad math corrupts training silently (loss spikes/NaNs hours later). |
| Throttle Reasons | Clocks Throttle Reasons | DCGM bitmask explaining *why* a GPU downclocked: thermal, power (TGP cap), HW slowdown, sync boost, etc. |
| Thermal Throttling | Thermal Throttling | Automatic downclock when the die or HBM exceeds a temperature limit. "Healthy but slow." |
| TDP / TGP | Thermal Design Power / Total Graphics Power | The GPU's power/thermal envelope. **Power capping** enforces it, reducing performance before a crash. |
| P-State | Performance State | GPU power/performance state (P0 = maximum performance … P12 = idle). |
| SMCLK / MCLK | SM Clock / Memory Clock | Core and memory clock frequencies. Sudden drops reveal throttling. |
| Power Draw | Board Power Draw | Instantaneous board watts; compared against TGP it shows capping headroom. |

## Reliability, SLO & Predictive Math

| Short Name | Full Name | Short Description |
|---|---|---|
| MTBF | Mean Time Between Failures | Average uptime between failures; a core fleet-reliability measure. |
| MTTR | Mean Time To Repair / Recovery | Average time to restore a failed node or GPU to service. |
| RUL | Remaining Useful Life | Predicted time until a degrading component fails — the target of predictive maintenance (a V2 capability). |
| Availability | Availability | Fraction of time a resource is usable, often MTBF ÷ (MTBF + MTTR). Underlies the SLA. |
| Error Budget | Error Budget | The amount of unreliability an SLO permits — how much you can "spend" before you must stop and fix. |

## Network Performance (complements the fabric terms above)

| Short Name | Full Name | Short Description |
|---|---|---|
| PFC | Priority Flow Control | Per-traffic-class pause on lossless Ethernet/RoCE. Misconfiguration causes stalls and head-of-line blocking. |
| ECN | Explicit Congestion Notification | Marks packets instead of dropping them, signaling senders to slow down (RoCE congestion control). |
| HoL Blocking | Head-of-Line Blocking | A stalled flow at the front of a queue holds up everything behind it. |
| Tail Latency | Tail Latency (p99 / p99.9) | The slow end of the latency distribution; dominates distributed-training stalls and inference SLOs. |
| Incast | Incast Congestion | Many senders targeting one receiver at once (e.g., AllReduce), overrunning switch/NIC buffers. |
