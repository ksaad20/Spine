# Spine OS — Platform Architecture Specification v0.1

**Document ID:** SPINE-ARCH-001  
**Version:** 0.1.0-alpha  
**Date:** 2026-07-31  
**Status:** Draft — Phase 1 Foundation  
**Author:** Spine Core Team  

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Core Abstraction Layers](#3-core-abstraction-layers)
4. [Component Model](#4-component-model)
5. [Communication Substrate](#5-communication-substrate)
6. [Resource & Lifecycle Management](#6-resource--lifecycle-management)
7. [Security Architecture](#7-security-architecture)
8. [Extensibility & Plugin Framework](#8-extensibility--plugin-framework)
9. [Deployment Topology](#9-deployment-topology)
10. [Glossary](#10-glossary)

---

## 1. Design Philosophy

Spine is designed as a **next-generation robot operating system ecosystem** that addresses the fundamental limitations of legacy middleware. The following principles guide every architectural decision:

### 1.1 First Principles

| Principle | Rationale | Implication |
|-----------|-----------|-------------|
| **Deterministic by Default** | Robotics requires predictable timing. Non-deterministic behavior must be opt-in, not the default. | Core scheduling, message passing, and resource allocation are deterministic. Best-effort paths are explicitly marked. |
| **Zero-Copy Everywhere** | Memory bandwidth is the bottleneck in high-frequency sensor pipelines. | The communication layer supports true zero-copy shared memory for intra-host transport. |
| **Heterogeneous by Design** | Modern robots combine microcontrollers, edge AI accelerators, and cloud backends. | The architecture treats a 32-bit MCU and a 64-core server as first-class citizens in the same logical topology. |
| **Fail-Operational, Not Fail-Safe** | Robots cannot always stop safely. Degradation must be graceful and bounded. | Every component defines a degradation contract with explicit fallback behaviors. |
| **Composability over Monoliths** | Reuse beats rewrite. Components must compose without recompilation. | Strict interface contracts and dynamic loading are architectural requirements, not implementation details. |

### 1.2 Non-Goals

To maintain focus, the following are explicitly **not** goals of the Spine platform architecture:

- **Real-time guarantees on non-real-time OS kernels.** Spine provides deterministic primitives; the host kernel must cooperate.
- **Replacing safety-certified automotive or aerospace stacks.** Spine targets research, prototyping, and commercial robotics where rapid iteration is valued over certification.
- **Cloud-only operation.** Spine works offline; cloud integration is an extension, not a dependency.

---

## 2. System Architecture Overview

Spine adopts a **layered microkernel-inspired architecture**. Unlike a pure microkernel, performance-critical paths (communication, scheduling) are partially kernelized to avoid the overhead of full user-space mediation.

### 2.1 High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Perception │  │   Planning  │  │  Control    │  │  Human-Robot        │  │
│  │  Pipelines  │  │   Engines   │  │  Loops      │  │  Interaction        │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
└─────────┼────────────────┼────────────────┼────────────────────┼─────────────┘
          │                │                │                    │
          ▼                ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SPINE RUNTIME LAYER                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    COMPONENT FRAMEWORK (CFW)                           │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │  Node    │  │  Service │  │  Action  │  │  Event   │  │  Param   │  │ │
│  │  │  Manager │  │  Router  │  │  Server  │  │  Bus     │  │  Server  │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │              COMMUNICATION SUBSTRATE (COMMS)                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  Shared Mem  │  │  DDS/XRCE    │  │  IPC Bridge  │  │  Network     │  │ │
│  │  │  Transport   │  │  Transport   │  │  Transport   │  │  Discovery   │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │           EXECUTION MANAGEMENT (EXEC)                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  Scheduler   │  │  Executor    │  │  Watchdog  │  │  Profiler    │  │ │
│  │  │  (Determ.)   │  │  (Work Steal)│  │  (Health)    │  │  (Telemetry) │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │                │                │                    │
          ▼                ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HARDWARE ABSTRACTION LAYER (HAL)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Sensor      │  │  Actuator    │  │  Compute     │  │  Comm        │       │
│  │  Drivers     │  │  Drivers     │  │  Backends    │  │  Drivers     │       │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
          │                │                │                    │
          ▼                ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HARDWARE LAYER                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  LiDAR   │  │  Motors  │  │  Cameras │  │  IMU     │  │  Network │        │
│  │  (Ether) │  │  (CAN)   │  │  (MIPI)  │  │  (I2C)   │  │  (WiFi)  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Layer Responsibilities

| Layer | Responsibility | Guarantees |
|-------|---------------|------------|
| **Application** | Domain-specific algorithms (SLAM, grasp planning, speech). | No direct hardware access. Composes via Spine Runtime APIs only. |
| **Spine Runtime** | Component lifecycle, communication, execution, and resource arbitration. | Deterministic scheduling contracts, typed message interfaces, fault isolation. |
| **HAL** | Normalize hardware into typed device interfaces. | Every device exposes the same API surface regardless of vendor or bus. |
| **Hardware** | Physical sensors, actuators, and compute units. | Opaque to upper layers; HAL owns the translation. |

---

## 3. Core Abstraction Layers

### 3.1 The Component Framework (CFW)

The CFW is the **soul of Spine**. It defines how software units (called **Nodes**) are created, composed, and executed.

#### 3.1.1 Node Model

A **Node** is the atomic unit of computation in Spine. It is:

- **Single-threaded by default** — one event loop, one owner thread.
- **Message-driven** — all inputs arrive via typed ports; outputs leave via typed ports.
- **State-machine-managed** — every node exposes a lifecycle state machine (Unconfigured → Inactive → Active → Finalized).

```
┌─────────────────────────────────────────┐
│              SPINE NODE                 │
│  ┌─────────────────────────────────┐    │
│  │         NODE STATE MACHINE       │    │
│  │  [Unconfigured] ──configure()──► │    │
│  │       │                          │    │
│  │       ▼ activate()               │    │
│  │   [Inactive] ─────────────────►  │    │
│  │       │         [Active]         │    │
│  │       ▼ deactivate()             │    │
│  │   [Inactive] ◄────────────────── │    │
│  │       │ cleanup()                │    │
│  │       ▼                          │    │
│  │   [Finalized]                    │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ Input   │  │ Output  │  │ Parameter│  │
│  │ Ports   │  │ Ports   │  │ Interface│  │
│  │ (typed) │  │ (typed) │  │ (dynamic)│  │
│  └─────────┘  └─────────┘  └─────────┘  │
└─────────────────────────────────────────┘
```

#### 3.1.2 Port Semantics

| Port Type | Direction | Cardinality | Delivery Guarantee |
|-----------|-----------|-------------|-------------------|
| **Topic** | Input / Output | 1:N publisher, M:N subscriber | Best-effort, bounded loss |
| **Service** | Client / Server | 1:1 per call | At-least-once with timeout |
| **Action** | Client / Server | 1:1 per goal | At-least-once, preemptable, with feedback |
| **Event** | Input only | 1:1 per node | Exactly-once, ordered, system-level |

### 3.2 Communication Substrate (COMMS)

Spine does not mandate a single transport. Instead, it defines a **transport-agnostic message interface** and provides multiple backends selected at runtime based on deployment topology.

#### 3.2.1 Transport Matrix

| Topology | Default Transport | Fallback | Latency Target |
|----------|------------------|----------|----------------|
| Intra-process (same address space) | Shared memory ring buffer | — | < 1 µs |
| Intra-host (same machine, different process) | Shared memory + semaphore | Unix domain socket | < 10 µs |
| Inter-host (same subnet, wired) | eUDP (enhanced UDP) | TCP | < 100 µs |
| Inter-host (wireless / lossy) | eUDP + FEC | TCP + retry | < 5 ms |
| MCU bridge (microcontroller) | XRCE-DDS over serial | Custom lightweight protocol | < 1 ms |
| Cloud bridge | gRPC over QUIC | WebSocket | < 50 ms |

#### 3.2.2 Zero-Copy Contract

For intra-process and intra-host transports, Spine guarantees:

1. **Single allocation** — the message buffer is allocated once by the publisher.
2. **Reference counting** — subscribers receive a read-only view with automatic reclamation.
3. **No serialization** — messages are laid out in shared memory in their native wire format (flat, cache-line aligned).

This is enforced by the **Spine Message Format (SMF)**, a schema-driven binary format with the following properties:

- Fixed-size headers (cache-line aligned).
- Variable-length payloads with inline size fields.
- No pointers — only offsets.
- Cross-platform endianness (little-endian default, big-endian on legacy hardware).

### 3.3 Execution Management (EXEC)

#### 3.3.1 Scheduler Architecture

Spine uses a **hierarchical scheduler** with two tiers:

**Tier 1 — Deterministic Scheduler (DS)**
- Reserved for safety-critical and real-time components.
- Uses a **rate-monotonic** or **deadline-monotonic** policy.
- Requires components to declare their **period**, **deadline**, and **WCET** (worst-case execution time) at registration.
- Runs on isolated CPU cores when available (via `isolcpus` or equivalent).

**Tier 2 — Work-Stealing Scheduler (WS)**
- For best-effort and event-driven components.
- Uses a thread-pool with work-stealing queues.
- Automatically scales thread count based on load and core availability.
- Components can be **promoted** to Tier 1 at runtime if they begin exhibiting periodic behavior.

#### 3.3.2 Executor Model

Each Node is bound to exactly one **Executor**. An Executor is a scheduling context (thread or process) that:

- Collects incoming messages into a **wait set**.
- Triggers the Node's callback when the wait set is satisfied.
- Enforces callback **reentrancy policies**: mutually exclusive, reentrant, or serialized.

```
┌─────────────────────────────────────────────┐
│              EXECUTOR                        │
│  ┌───────────────────────────────────────┐  │
│  │  Wait Set                             │  │
│  │  [Topic A] [Topic B] [Timer 10ms]   │  │
│  └──────────────────┬────────────────────┘  │
│                     │ trigger                │
│                     ▼                        │
│  ┌───────────────────────────────────────┐  │
│  │  Node Callback (user code)            │  │
│  │  • Read inputs                        │  │
│  │  • Compute                            │  │
│  │  • Write outputs                      │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

---

## 4. Component Model

### 4.1 Node Composition Patterns

Spine supports three composition patterns, selected at deployment time without code changes:

#### 4.1.1 Composition by Container (Process Isolation)
Each Node runs in its own OS process. Maximum fault isolation. Higher latency due to IPC.

#### 4.1.2 Composition by Shared Object (Thread Isolation)
Multiple Nodes are loaded into a single process as shared libraries. Lower latency via shared memory. A crash in one Node crashes the entire process.

#### 4.1.3 Composition by Inlining (Zero Isolation)
Nodes are statically linked into a single binary. Used for deeply embedded deployments. No dynamic loading.

### 4.2 Component Interface Definition Language (CIDL)

All Node interfaces are declared in **CIDL**, a schema language that generates:

- Message structs (C++, Python, Rust, MCU C).
- Serialization / deserialization code.
- Client / server stubs for services and actions.
- Documentation and validation rules.

Example CIDL:

```cidl
package spine.sensor;

message ImuReading {
    timestamp_ns: u64;
    frame_id: string[64];

    struct AngularVelocity {
        x: f64;
        y: f64;
        z: f64;
    }

    struct LinearAcceleration {
        x: f64;
        y: f64;
        z: f64;
    }

    angular_velocity: AngularVelocity;
    linear_acceleration: LinearAcceleration;
    covariance: f64[36];  // Row-major 6x6
}

service ImuDriver {
    method calibrate(offset: CalibrationConfig) -> CalibrationResult;
    method set_rate(hz: u16) -> Result;
}
```

---

## 5. Communication Substrate

### 5.1 Discovery Protocol

Spine uses a **distributed gossip-based discovery** protocol with no central name server.

- Each Node advertises its **topic subscriptions**, **service offerings**, and **action capabilities** via UDP multicast on the local subnet.
- For cross-subnet or cloud deployments, a **Discovery Relay** bridges gossip domains.
- Discovery state converges in **O(log N)** rounds for N nodes under normal churn.
- Nodes cache discovery entries with a TTL. Stale entries are purged automatically.

### 5.2 Quality of Service (QoS) Profiles

Spine defines **six standard QoS profiles** that cover 95% of robotics use cases:

| Profile | Reliability | Durability | History | Deadline | Use Case |
|---------|-------------|------------|---------|----------|----------|
| **Sensor** | Best-effort | Volatile | Keep-last (1) | 10 ms | High-frequency sensor streams (LiDAR, camera) |
| **Telemetry** | Best-effort | Volatile | Keep-last (10) | 100 ms | Diagnostics, logging, metrics |
| **Command** | Reliable | Volatile | Keep-last (1) | 50 ms | Control setpoints, motion commands |
| **State** | Reliable | Transient-local | Keep-last (1) | 1 s | System state, mode changes |
| **Parameter** | Reliable | Transient-local | Keep-all | ∞ | Configuration, calibration |
| **Critical** | Reliable | Transient-local | Keep-last (1) | 5 ms | Emergency stop, safety limits |

Users may define custom profiles, but standard profiles are **pre-optimized** in each transport backend.

### 5.3 Time Synchronization

Spine requires a ** unified time domain** across all nodes in a deployment.

- **Primary clock:** `spine::Time` — a 64-bit nanosecond count from an arbitrary epoch (boot time of the oldest node).
- **Synchronization:** PTP (IEEE 1588) for wired networks; NTP fallback for wireless/cloud.
- **Message timestamps:** Every message carries a `timestamp_ns` field. The communication layer does **not** stamp messages automatically — the producer is responsible for timestamping at the moment of acquisition or generation.
- **Latency reporting:** The communication layer optionally records `ingress_ns` and `egress_ns` in message metadata for latency profiling.

---

## 6. Resource & Lifecycle Management

### 6.1 Node Lifecycle State Machine

Every Node must implement the following state machine. The Spine Runtime enforces transitions; user code provides callbacks.

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
┌─────────────┐  configure()  ┌──────────┐  activate()  ┌─────────┐
│ Unconfigured│ ─────────────►│ Inactive │ ────────────►│ Active  │
└─────────────┘               └────┬─────┘              └────┬────┘
     ▲                             │                       │
     │                             │ deactivate()          │
     │ cleanup()                   │ ◄─────────────────────┘
     │ ◄─────────────────────────────┘
     │
     └──────────────────────────────────────────────────────────┘
```

| State | Allowed Operations | User Callback |
|-------|-------------------|---------------|
| **Unconfigured** | `configure()` | `on_configure()` — allocate resources, declare ports, validate parameters. |
| **Inactive** | `activate()`, `cleanup()` | `on_activate()` — start periodic timers, open hardware handles. |
| **Active** | `deactivate()` | `on_deactivate()` — stop timers, flush buffers, release hardware. |
| **Finalized** | None | `on_cleanup()` — free all resources, close file handles. |

### 6.2 System-Level Lifecycle

Above individual Nodes, Spine defines a **System Manifest** — a declarative file that describes:

- Which Nodes to launch.
- Their interconnections (topic wiring).
- Their resource budgets (CPU affinity, memory limits, real-time priority).
- Their launch order and dependencies.

The **System Manager** reads the manifest and orchestrates the entire lifecycle:

1. **Validation** — checks that all declared topics have at least one publisher and one subscriber.
2. **Scheduling** — assigns Nodes to executors and executors to CPU cores.
3. **Launch** — brings Nodes to `Active` in dependency order.
4. **Monitoring** — watches health heartbeats and triggers degradation on failure.
5. **Shutdown** — brings Nodes to `Finalized` in reverse dependency order.

### 6.3 Resource Budgeting

Every Node declares a **Resource Budget** in its manifest:

```yaml
node:
  name: lidar_driver
  budget:
    cpu_cores: [2, 3]          # Preferred affinity
    memory_mb: 128             # Soft limit; OOM killer trigger
    realtime_priority: 50      # SCHED_FIFO priority
    deadline_us: 5000          # 5 ms callback deadline
    wcet_us: 2000              # 2 ms worst-case execution time
```

The System Manager uses these budgets to:
- Allocate CPU cores via `isolcpus` / `cpuset`.
- Set `ulimit` and `cgroups` memory constraints.
- Configure `SCHED_FIFO` / `SCHED_DEADLINE`.
- Validate schedulability via a **utilization bound test** before launch.

---

## 7. Security Architecture

### 7.1 Threat Model

Spine assumes the following threat model:

- **Trusted compute host** — the machine running Spine Runtime is physically secured.
- **Untrusted network** — any network outside the robot's physical chassis is untrusted.
- **Semi-trusted components** — Nodes from third-party vendors are sandboxed and capability-restricted.

### 7.2 Security Layers

| Layer | Mechanism | Scope |
|-------|-----------|-------|
| **Transport Encryption** | TLS 1.3 for TCP/gRPC; DTLS for UDP; AES-GCM for shared memory in multi-tenant environments. | Inter-host and cloud traffic. |
| **Authentication** | X.509 certificates per Node; SPIFFE/SPIRE for dynamic attestation. | All Nodes in a deployment. |
| **Authorization** | Capability-based access control (CBAC). Every port has an ACL. | Topic publish, subscribe, service call. |
| **Sandboxing** | seccomp-bpf + Landlock for process-isolated Nodes; namespace isolation for shared-object Nodes. | Third-party and untrusted Nodes. |
| **Audit** | Structured logging of all security-relevant events (auth, ACL violations, lifecycle transitions). | System-wide. |

### 7.3 Secure Bootstrapping

When a new Node joins a Spine deployment:

1. **Attestation** — the Node presents its certificate to the System Manager.
2. **Authorization** — the System Manager checks the Node's manifest against the deployment's policy.
3. **Capability Grant** — on success, the Node receives a **capability token** that encodes its allowed ports and operations.
4. **Token Validation** — every message and service call carries the token; the transport layer validates it at ingress.

---

## 8. Extensibility & Plugin Framework

### 8.1 Plugin Types

Spine supports four categories of plugins, each loaded dynamically by the Runtime:

| Plugin Type | Interface | Loaded By | Example |
|-------------|-----------|-----------|---------|
| **Transport Plugin** | `spine::transport::Backend` | COMMS | A custom CAN-FD transport for automotive robots. |
| **Executor Plugin** | `spine::exec::Executor` | EXEC | A GPU-based executor for CUDA-accelerated perception nodes. |
| **HAL Plugin** | `spine::hal::DeviceDriver` | HAL | A vendor-specific motor controller driver. |
| **Tooling Plugin** | `spine::tool::Analyzer` | CLI | A custom latency analyzer for the `spine doctor` command. |

### 8.2 Plugin Contract

Every plugin exports a single C ABI function:

```c
spine_plugin_descriptor_t* spine_plugin_init(void);
```

The descriptor contains:
- Plugin name and version.
- Required Runtime version.
- Exported interface pointers.
- Resource requirements (memory, threads, file descriptors).

The Runtime validates compatibility at load time and sandboxes the plugin according to its declared requirements.

### 8.3 Plugin Registry

Plugins are discovered via:

1. **Filesystem scan** — the Runtime searches `SPINE_PLUGIN_PATH` (default: `/usr/lib/spine/plugins/`).
2. **Manifest declaration** — the System Manifest can explicitly list plugins to load.
3. **Lazy loading** — plugins are loaded on first use unless the manifest requests eager loading.

---

## 9. Deployment Topology

### 9.1 Single-Host Deployment

All Nodes run on a single compute unit (e.g., an NVIDIA Jetson or x86 IPC).

- **Transport:** Shared memory for intra-process; Unix domain sockets for inter-process.
- **Discovery:** Local multicast (loopback only).
- **Use case:** Small mobile robots, desktop simulation, development.

### 9.2 Multi-Host Deployment

Nodes are distributed across multiple compute units connected by Ethernet or WiFi.

- **Transport:** eUDP for topics; gRPC for services/actions.
- **Discovery:** Multicast on each subnet + Discovery Relay between subnets.
- **Time sync:** PTP across wired links; NTP for wireless segments.
- **Use case:** Humanoid robots (separate head and torso computers), multi-robot fleets.

### 9.3 Edge-Cloud Deployment

Some Nodes run on the robot; others run in the cloud.

- **Transport:** QUIC for topics; gRPC for services.
- **Discovery:** Cloud Discovery Relay with TLS mutual auth.
- **Bandwidth management:** Automatic topic down-sampling and compression for cloud-bound streams.
- **Use case:** Fleet learning, remote teleoperation, heavy compute offloading.

### 9.4 MCU Bridge Deployment

A microcontroller (ARM Cortex-M, ESP32, RP2040) runs lightweight Nodes bridged to the main Spine deployment.

- **Transport:** XRCE-DDS over UART/SPI/CAN-FD.
- **Footprint:** < 64 KB RAM, < 256 KB flash for the Spine MCU runtime.
- **Use case:** Low-level motor control, safety-critical emergency stops, sensor preprocessing.

---

## 10. Glossary

| Term | Definition |
|------|-----------|
| **Node** | The atomic unit of computation in Spine. A single-threaded, message-driven component with a lifecycle state machine. |
| **Topic** | A named, typed message bus supporting many-to-many publish-subscribe semantics. |
| **Service** | A synchronous request-response interface between exactly one client and one server per call. |
| **Action** | An asynchronous, preemptable, long-running task with feedback and result. |
| **Executor** | A scheduling context (thread or process) that triggers Node callbacks based on a wait set. |
| **HAL** | Hardware Abstraction Layer. Normalizes physical devices into typed, portable interfaces. |
| **SMF** | Spine Message Format. A schema-driven, zero-copy binary message layout. |
| **CIDL** | Component Interface Definition Language. Declares messages, services, and actions. |
| **System Manifest** | A declarative file describing a complete Spine deployment: nodes, wiring, and resource budgets. |
| **QoS Profile** | A pre-defined combination of reliability, durability, history, and deadline settings. |
| **Discovery Relay** | A bridge that propagates node discovery information across network boundaries. |
| **Capability Token** | A cryptographically signed token encoding a Node's authorized ports and operations. |

---

## Appendix A: Design Decision Records

### ADR-001: Why Not DDS Native?

**Context:** Many robotics systems use DDS (Data Distribution Service) as the communication backbone.  
**Decision:** Spine defines its own transport abstraction (`COMMS`) and uses DDS (specifically eProsima Fast-DDS with XRCE) only as one of several backends.  
**Rationale:**
- DDS is overkill for intra-process communication.
- DDS discovery is chatty and slow to converge on large networks.
- DDS's QoS model is powerful but confusing; Spine's six standard profiles are simpler.
- DDS does not natively support MCU bridging (XRCE is an add-on).

**Consequences:**
- We maintain a thin abstraction layer.
- We gain the ability to swap DDS for a lighter protocol in resource-constrained deployments.

### ADR-002: Why Shared Memory Over Message Queues?

**Context:** POSIX message queues and pipes are standard IPC mechanisms.  
**Decision:** Spine uses lock-free shared memory ring buffers for intra-host transport.  
**Rationale:**
- Message queues have kernel-copy overhead and size limits.
- Shared memory allows true zero-copy for large messages (point clouds, images).
- Lock-free algorithms (SPSC/MPSC ring buffers) avoid kernel context switches.

**Consequences:**
- Requires careful memory management and reference counting.
- Crash of a producer can leave consumers with dangling references (mitigated by watchdog).

### ADR-003: Why CIDL Instead of Protobuf/FlatBuffers?

**Context:** Existing IDLs like Protobuf, FlatBuffers, and Cap'n Proto are mature.  
**Decision:** Spine defines CIDL, a robotics-specific IDL.  
**Rationale:**
- Protobuf requires serialization/deserialization (not zero-copy).
- FlatBuffers supports zero-copy but lacks service/action semantics.
- Neither has first-class support for real-time constraints (no heap allocation, bounded sizes).
- CIDL can generate MCU-compatible C code with deterministic memory layouts.

**Consequences:**
- We maintain a compiler toolchain.
- Users learn a new IDL, but it is simpler than Protobuf for robotics use cases.

---

*End of Document*
