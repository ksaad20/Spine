# Spine OS — Technical Standards v0.1

**Document ID:** SPINE-STD-001  
**Version:** 0.1.0-alpha  
**Date:** 2026-07-31  
**Status:** Draft — Phase 1 Foundation  
**Supersedes:** N/A  
**Author:** Spine Core Team  
**License:** Apache-2.0  

---

## Abstract

Spine is a next-generation robot operating system ecosystem designed to address the fundamental limitations of legacy robotics middleware. This document defines the complete technical standard for Spine OS Phase 1, encompassing platform architecture, hardware abstraction, driver interfaces, and publication protocols. Spine prioritizes deterministic execution, zero-copy communication, hardware interchangeability, and graceful degradation. These standards establish the contractual foundation for all Spine-compliant implementations, tools, and integrations.

**Keywords:** robot operating system, hardware abstraction, real-time middleware, deterministic scheduling, zero-copy communication, robotics driver model, component framework

---

## Table of Contents

1. [Normative References](#1-normative-references)
2. [Terms and Definitions](#2-terms-and-definitions)
3. [System Architecture Standard](#3-system-architecture-standard)
4. [Hardware Abstraction Layer Standard](#4-hardware-abstraction-layer-standard)
5. [Driver Interface Standard](#5-driver-interface-standard)
6. [Message Format Standard](#6-message-format-standard)
7. [Configuration and Calibration Standard](#7-configuration-and-calibration-standard)
8. [Safety and Real-Time Standard](#8-safety-and-real-time-standard)
9. [Conformance and Validation](#9-conformance-and-validation)
10. [Versioning and Evolution](#10-versioning-and-evolution)
11. [Publication Protocol](#11-publication-protocol)
12. [Appendices](#12-appendices)

---

## 1. Normative References

The following documents are referenced in this standard and constitute requirements:

| Reference ID | Document | Version | Authority |
|-------------|----------|---------|-----------|
| SPINE-ARCH-001 | Spine OS — Platform Architecture Specification | 0.1.0-alpha | Spine Core Team |
| SPINE-HAL-001 | Spine OS — Hardware Abstraction Layer Specification | 0.1.0-alpha | Spine Core Team |
| SPINE-DRV-001 | Spine OS — Driver Specification | 0.1.0-alpha | Spine Core Team |
| ISO/IEC 15026 | Systems and software engineering — Systems and software assurance | 2019 | ISO |
| IEC 61508 | Functional safety of electrical/electronic/programmable electronic safety-related systems | 2010 | IEC |
| IEEE 1588 | Precision Clock Synchronization Protocol for Networked Measurement and Control Systems | 2019 | IEEE |
| OMG DDS | Data Distribution Service (DDS) Specification | v1.4 | OMG |

---

## 2. Terms and Definitions

For the purposes of this document, the following terms and definitions apply:

| Term | Definition |
|------|-----------|
| **Spine Node** | The atomic unit of computation in Spine. A single-threaded, message-driven component with a four-state lifecycle machine (Unconfigured, Inactive, Active, Finalized). |
| **Spine Runtime** | The execution environment that manages component lifecycle, communication, scheduling, and resource arbitration. |
| **HAL** | Hardware Abstraction Layer. The boundary between the Spine Runtime and physical hardware. |
| **HAL-API** | The service interface exposed by HAL to the Runtime: Device Registry, Config Server, Diagnostic Hub, and Event Notifier. |
| **Driver** | A Spine Node that manages a specific hardware device via a bus adapter. |
| **CIDL** | Component Interface Definition Language. The schema-driven language for defining messages, services, and actions. |
| **SMF** | Spine Message Format. A binary, zero-copy message layout with cache-line-aligned headers. |
| **QoS Profile** | A pre-defined combination of reliability, durability, history, and deadline settings for communication. |
| **Zero-Copy** | A transport optimization where message data is shared via shared memory without serialization or memory copying. |
| **Calibration** | The process of determining intrinsic, extrinsic, temporal, or dynamic parameters that map raw device output to physically meaningful data. |
| **E-Stop** | Emergency Stop. A system-level safety mechanism that immediately disables all actuators. |
| **WCET** | Worst-Case Execution Time. The maximum time a callback may take under any conditions. |
| **Deterministic Scheduler** | The Spine scheduler tier reserved for safety-critical and real-time components using rate-monotonic or deadline-monotonic policies. |
| **Work-Stealing Scheduler** | The Spine scheduler tier for best-effort and event-driven components using a dynamically scaling thread pool. |
| **Bus Adapter** | The HAL component that abstracts a specific physical bus into the generic BusAdapter interface. |
| **Device Descriptor** | The metadata record for every hardware instance, including class, family, vendor, model, bus, and health status. |
| **Driver Manifest** | The declarative YAML file that describes a driver's capabilities, requirements, ports, and metadata. |
| **System Manifest** | The declarative file that describes a complete Spine deployment: nodes, wiring, and resource budgets. |
| **Capability Token** | A cryptographically signed token encoding a Node's authorized ports and operations. |
| **Complexity Tier** | Driver classification: T1 (Simple), T2 (Streaming), T3 (Closed-Loop), T4 (Safety-Critical). |

---

## 3. System Architecture Standard

### 3.1 Layered Architecture

Spine shall implement a four-layer architecture:

```
+-------------------------------------------------------------+
|  LAYER 4: APPLICATION                                       |
|  Domain-specific algorithms and user-facing logic.          |
|  No direct hardware access permitted.                       |
+-------------------------------------------------------------+
                            |
                            | Spine Runtime API
                            v
+-------------------------------------------------------------+
|  LAYER 3: SPINE RUNTIME                                   |
|  Component Framework (CFW)                                  |
|  Communication Substrate (COMMS)                            |
|  Execution Management (EXEC)                                |
+-------------------------------------------------------------+
                            |
                            | HAL-API
                            v
+-------------------------------------------------------------+
|  LAYER 2: HARDWARE ABSTRACTION LAYER (HAL)                  |
|  HAL Core: Device Manager, Bus Router, Config Manager,        |
|            Safety Monitor                                   |
|  Driver Framework: Sensor, Actuator, Compute, Comm drivers |
|  Bus Adapters: CAN, EtherCAT, I2C, SPI, UART, USB, Ethernet, |
|                MIPI, PCIe                                   |
+-------------------------------------------------------------+
                            |
                            | Kernel Drivers / Raw Protocols
                            v
+-------------------------------------------------------------+
|  LAYER 1: HARDWARE                                          |
|  Physical sensors, actuators, compute units, and networks.  |
+-------------------------------------------------------------+
```

**Normative Requirement (ARCH-001):** Application-layer code shall not access hardware directly. All hardware interaction shall be mediated by the HAL.

### 3.2 Design Principles

The following principles are normative for all Spine implementations:

| ID | Principle | Requirement |
|----|-----------|-------------|
| ARCH-P1 | Deterministic by Default | Core scheduling, message passing, and resource allocation shall be deterministic. Best-effort paths shall be explicitly marked. |
| ARCH-P2 | Zero-Copy Everywhere | Intra-process and intra-host transports shall support true zero-copy shared memory. Single allocation, reference counting, no serialization. |
| ARCH-P3 | Heterogeneous by Design | A 32-bit MCU and a 64-core server shall be first-class citizens in the same logical topology. |
| ARCH-P4 | Fail-Operational | Every component shall define a degradation contract with explicit fallback behaviors. |
| ARCH-P5 | Composability over Monoliths | Components shall compose without recompilation. Strict interface contracts and dynamic loading are required. |

### 3.3 Component Framework

#### 3.3.1 Node Lifecycle

Every Node shall implement the following state machine:

```
+-------------------------------------------------------------+
|                  NODE LIFECYCLE STATE MACHINE               |
|                                                             |
|   +------------------+                                     |
|   |   UNCONFIGURED   |                                     |
|   +--------+---------+                                     |
|            |                                                 |
|            | configure()                                    |
|            |  - on_configure() callback                     |
|            v                                                 |
|   +------------------+                                     |
|   |     INACTIVE     |                                     |
|   +--------+---------+                                     |
|            |                                                 |
|            | activate()                                     |
|            |  - on_activate() callback                      |
|            v                                                 |
|   +------------------+                                     |
|   |      ACTIVE      |                                     |
|   +--------+---------+                                     |
|            |                                                 |
|            | deactivate()                                   |
|            |  - on_deactivate() callback                    |
|            v                                                 |
|   +------------------+                                     |
|   |     INACTIVE     |                                     |
|   +--------+---------+                                     |
|            |                                                 |
|            | cleanup()                                      |
|            |  - on_cleanup() callback                       |
|            v                                                 |
|   +------------------+                                     |
|   |    FINALIZED     |                                     |
|   +------------------+                                     |
|                                                             |
+-------------------------------------------------------------+
```

**Normative Requirement (ARCH-002):** State transitions shall be atomic and observable via the Event Bus. No state shall be skipped.

#### 3.3.2 Port Semantics

| Port Type | Direction | Cardinality | Delivery Guarantee |
|-----------|-----------|-------------|-------------------|
| Topic | Input / Output | 1:N publisher, M:N subscriber | Best-effort, bounded loss |
| Service | Client / Server | 1:1 per call | At-least-once with timeout |
| Action | Client / Server | 1:1 per goal | At-least-once, preemptable, with feedback |
| Event | Input only | 1:1 per node | Exactly-once, ordered, system-level |

### 3.4 Communication Substrate

#### 3.4.1 Transport Selection Matrix

| Topology | Default Transport | Fallback | Latency Target |
|----------|------------------|----------|----------------|
| Intra-process | Shared memory ring buffer | — | < 1 us |
| Intra-host | Shared memory + semaphore | Unix domain socket | < 10 us |
| Inter-host (wired) | eUDP | TCP | < 100 us |
| Inter-host (wireless) | eUDP + FEC | TCP + retry | < 5 ms |
| MCU bridge | XRCE-DDS over serial | Custom lightweight | < 1 ms |
| Cloud bridge | gRPC over QUIC | WebSocket | < 50 ms |

**Normative Requirement (ARCH-003):** Intra-process transport shall be zero-copy with no serialization overhead.

#### 3.4.2 QoS Profiles

| Profile | Reliability | Durability | History | Deadline | Use Case |
|---------|-------------|------------|---------|----------|----------|
| Sensor | Best-effort | Volatile | Keep-last (1) | 10 ms | High-frequency sensor streams |
| Telemetry | Best-effort | Volatile | Keep-last (10) | 100 ms | Diagnostics, logging |
| Command | Reliable | Volatile | Keep-last (1) | 50 ms | Control setpoints |
| State | Reliable | Transient-local | Keep-last (1) | 1 s | System state |
| Parameter | Reliable | Transient-local | Keep-all | Inf | Configuration |
| Critical | Reliable | Transient-local | Keep-last (1) | 5 ms | Emergency stop |

### 3.5 Execution Management

#### 3.5.1 Hierarchical Scheduler

```
+-------------------------------------------------------------+
|              HIERARCHICAL SCHEDULER ARCHITECTURE            |
|                                                             |
|   TIER 1: DETERMINISTIC SCHEDULER (DS)                     |
|   +-----------------------------------------------------+   |
|   |  Policy: Rate-Monotonic / Deadline-Monotonic      |   |
|   |  Cores: Isolated (isolcpus)                         |   |
|   |  Components: Safety-critical, real-time control       |   |
|   |  Requirements: Period, Deadline, WCET declared        |   |
|   +-----------------------------------------------------+   |
|                            |                                |
|                            | promotion                      |
|                            v                                |
|   TIER 2: WORK-STEALING SCHEDULER (WS)                     |
|   +-----------------------------------------------------+   |
|   |  Policy: Thread-pool with work-stealing queues      |   |
|   |  Cores: Shared with OS and other processes            |   |
|   |  Components: Best-effort, event-driven                |   |
|   |  Scaling: Automatic based on load                     |   |
|   +-----------------------------------------------------+   |
|                                                             |
+-------------------------------------------------------------+
```

**Normative Requirement (ARCH-004):** The Deterministic Scheduler shall run on isolated CPU cores when available. SCHED_FIFO or SCHED_DEADLINE shall be used.

---

## 4. Hardware Abstraction Layer Standard

### 4.1 HAL Architecture

```
+-------------------------------------------------------------+
|                    HAL ARCHITECTURE                         |
|                                                             |
|  +-------------------------------------------------------+  |
|  |  HAL CORE                                              |  |
|  |  +--------+  +--------+  +--------+  +--------+       |  |
|  |  | Device |  |  Bus   |  | Config |  | Safety |       |  |
|  |  |Manager |  | Router |  |Manager |  |Monitor |       |  |
|  |  +--------+  +--------+  +--------+  +--------+       |  |
|  +-------------------------------------------------------+  |
|                                                             |
|  +-------------------------------------------------------+  |
|  |  DRIVER FRAMEWORK                                      |  |
|  |  +--------+  +--------+  +--------+  +--------+       |  |
|  |  | Sensor |  |Actuator|  |Compute |  |  Comm  |       |  |
|  |  | Driver |  | Driver |  | Driver |  | Driver |       |  |
|  |  +--------+  +--------+  +--------+  +--------+       |  |
|  +-------------------------------------------------------+  |
|                                                             |
|  +-------------------------------------------------------+  |
|  |  BUS ADAPTERS                                          |  |
|  |  +------+ +------+ +------+ +------+ +------+       |  |
|  |  | CAN  | |Ether | | I2C  | | SPI  | | UART |       |  |
|  |  +------+ +------+ +------+ +------+ +------+       |  |
|  |  +------+ +------+ +------+ +------+                 |  |
|  |  | USB  | | Ether| | MIPI | | PCIe |                 |  |
|  |  +------+ +------+ +------+ +------+                 |  |
|  +-------------------------------------------------------+  |
|                                                             |
+-------------------------------------------------------------+
```

### 4.2 Device Taxonomy

All hardware devices shall be classified according to the following hierarchy:

```
DEVICE
|
+-- SENSOR
|   +-- RangeFinder (LiDAR, Ultrasonic, ToF)
|   +-- Camera (RGB, Depth, Thermal)
|   +-- Inertial (IMU, Magnetometer, AHRS)
|   +-- ForceTorque
|   +-- Encoder
|   +-- Tactile
|   +-- Gas / Chemical
|   +-- Temperature
|   +-- GPS / GNSS
|
+-- ACTUATOR
|   +-- Motor (DC, BLDC, Stepper, Servo)
|   +-- Gripper (Parallel, Vacuum, Underactuated)
|   +-- Linear Actuator
|   +-- Pneumatic Valve
|   +-- LED / Indicator
|
+-- COMPUTE
|   +-- GPU
|   +-- NPU / TPU
|   +-- FPGA
|   +-- DSP
|
+-- COMM
    +-- Ethernet Switch
    +-- WiFi Module
    +-- Bluetooth Module
    +-- Cellular
    +-- LoRa / Sub-GHz
```

**Normative Requirement (HAL-001):** Every device shall expose exactly one HAL interface regardless of internal bus complexity.

### 4.3 Driver Interface Contract

#### 4.3.1 Base Interface

All HAL drivers shall implement:

```cpp
class DriverBase : public core::Node {
    virtual Result on_configure(const core::NodeConfig& config) = 0;
    virtual Result on_activate() = 0;
    virtual Result on_deactivate() = 0;
    virtual Result on_cleanup() = 0;
    virtual hal::sensor::SensorDiagnostics get_diagnostics() = 0;
    virtual hal::sensor::SelfTestResult self_test() = 0;
    virtual hal::DeviceDescriptor get_descriptor() const = 0;
    virtual void on_heartbeat() = 0;
};
```

#### 4.3.2 Sensor Driver

```cpp
class SensorDriver : public DriverBase, public hal::sensor::SensorBase {
    virtual Result start_streaming(const hal::sensor::StreamingConfig& config) = 0;
    virtual Result stop_streaming() = 0;
    virtual hal::sensor::SensorReading get_reading() = 0;
    virtual hal::sensor::StreamingConfig get_default_streaming_config() const = 0;
};
```

#### 4.3.3 Actuator Driver

```cpp
class ActuatorDriver : public DriverBase, public hal::actuator::ActuatorBase {
    virtual Result enable() = 0;
    virtual Result disable() = 0;
    virtual hal::actuator::ActuatorState get_state() = 0;
    virtual Result emergency_stop() = 0;
    virtual Result clear_estop() = 0;
};
```

#### 4.3.4 Motor Driver

```cpp
class MotorDriver : public ActuatorDriver, public hal::actuator::motor::MotorDriver {
    virtual Result set_mode(hal::actuator::motor::ControlMode mode) = 0;
    virtual Result command(const hal::actuator::motor::MotorCommand& cmd) = 0;
    virtual Result set_gains(const hal::actuator::motor::PidGains& gains) = 0;
    virtual hal::actuator::motor::PidGains get_gains() = 0;
    virtual Result home() = 0;
};
```

### 4.4 Device Lifecycle

```
+-------------------------------------------------------------+
|                  DEVICE LIFECYCLE SUBSTATES                 |
|                                                             |
|   ACTIVE                                                     |
|   +--+--+--+--+                                             |
|   |  |  |  |  |                                             |
|   v  v  v  v  v                                             |
| +------+ +------+ +------+ +------+                         |
| |STREAM| |CALIB | |DEGRAD| |FAULT |                         |
| |ING   | |RAT-  | |ED    | |      |                         |
| |      | |ING   | |      | |      |                         |
| +------+ +------+ +------+ +------+                         |
|                                                             |
|   OFFLINE (reconnects to INACTIVE)                         |
|                                                             |
+-------------------------------------------------------------+
```

**Normative Requirement (HAL-002):** A device in the FAULT state shall not automatically transition to ACTIVE. Manual intervention or explicit re-initialization is required.

### 4.5 Calibration Framework

Calibration shall be classified into four types:

| Type | Description | Persistence |
|------|-------------|-------------|
| Intrinsic | Internal device parameters | Device firmware or driver storage |
| Extrinsic | Spatial transform to robot frame | Spine Parameter Server |
| Temporal | Clock offset from unified time | Runtime only |
| Dynamic | Temperature compensation, drift | Continuous update; statistics persisted |

**Normative Requirement (HAL-003):** Devices requiring calibration shall not enter the STREAMING state while `calibration_status == UNCALIBRATED`.

---

## 5. Driver Interface Standard

### 5.1 Driver Classification

Drivers shall be classified by Complexity Tier:

| Tier | Description | Real-Time | Calibration | Safety |
|------|-------------|-----------|-------------|--------|
| T1 — Simple | Polled, no stream | Soft | Optional | No |
| T2 — Streaming | Continuous data | Soft | Optional | No |
| T3 — Closed-Loop | RT control | Hard | Required | Optional |
| T4 — Safety-Critical | Human-safety | Hard | Required | Mandatory |

### 5.2 Naming Convention

Driver identifiers shall follow the URI schema:

```
spine://driver/{vendor}/{family}/{model}[/{variant}]
```

**Normative Requirement (DRV-001):** Driver IDs shall be globally unique within a Spine deployment.

### 5.3 Driver Manifest

Every driver shall ship a `driver.manifest.yaml` conforming to schema `spine.driver.manifest.v1`.

### 5.4 Timing Constraints

| Operation | T1 | T2 | T3 | T4 |
|-----------|-----|-----|-----|-----|
| `on_configure()` | 5 s | 5 s | 5 s | 5 s |
| `on_activate()` | 2 s | 2 s | 2 s | 2 s |
| `start_streaming()` | — | 100 ms | 100 ms | 50 ms |
| `emergency_stop()` | — | — | 10 ms | 1 ms |
| Heartbeat response | 200 ms | 200 ms | 20 ms | 10 ms |

---

## 6. Message Format Standard

### 6.1 Spine Message Format (SMF)

All Spine messages shall use the following header:

```
+-------------------------------------------------------------+
|              SMF MESSAGE HEADER (32 bytes)                  |
|                                                             |
|   +--------+--------+--------+--------+                     |
|   | Magic  |Version |MsgType |Reserved|                     |
|   | 4B     | 2B     | 2B     | 4B     |                     |
|   +--------+--------+--------+--------+                     |
|   | Payload Size (4B)                                       |
|   +---------------------------+                             |
|   | Checksum CRC32 (4B)      |                             |
|   +---------------------------+                             |
|   | Timestamp ns (8B)                                        |
|   +---------------------------+                             |
|   | Sequence (4B) | Flags (4B)                              |
|   +---------------------------+                             |
|                                                             |
|   Magic = 0x5350494E ("SPIN")                              |
|                                                             |
+-------------------------------------------------------------+
```

**Normative Requirement (MSG-001):** SMF headers shall be cache-line aligned (64-byte boundary). Padding shall be zero-filled.

### 6.2 Zero-Copy Contract

For intra-process and intra-host transports:

1. **Single allocation** — the message buffer is allocated once by the publisher.
2. **Reference counting** — subscribers receive a read-only view with automatic reclamation.
3. **No serialization** — messages are laid out in shared memory in native wire format.
4. **Endianness** — little-endian default; big-endian on legacy hardware only.

---

## 7. Configuration and Calibration Standard

### 7.1 Three-Tier Configuration Model

```
+-------------------------------------------------------------+
|              CONFIGURATION HIERARCHY                         |
|                                                             |
|   TIER 1: VENDOR DEFAULTS                                   |
|   +-----------------------------------------------------+   |
|   |  Shipped with driver. Read-only.                    |   |
|   +-----------------------------------------------------+   |
|                      |                                      |
|                      | override                             |
|                      v                                      |
|   TIER 2: DEPLOYMENT CONFIG                                 |
|   +-----------------------------------------------------+   |
|   |  Loaded from System Manifest. Overrides defaults.   |   |
|   +-----------------------------------------------------+   |
|                      |                                      |
|                      | override                             |
|                      v                                      |
|   TIER 3: RUNTIME PARAMETERS                                |
|   +-----------------------------------------------------+   |
|   |  Set via Parameter Server. Not persisted by default.|   |
|   +-----------------------------------------------------+   |
|                                                             |
+-------------------------------------------------------------+
```

### 7.2 Calibration State Machine

```
+-------------------------------------------------------------+
|              CALIBRATION STATE MACHINE                      |
|                                                             |
|   +-------------+                                           |
|   | UNCALIBRATED|                                           |
|   +------+------+                                           |
|          |                                                  |
|          | start_calibration()                             |
|          v                                                  |
|   +-------------+                                           |
|   | CALIBRATING |                                           |
|   +------+------+                                           |
|          |                                                  |
|          | success                                          |
|          v                                                  |
|   +-------------+                                           |
|   |  CALIBRATED |                                           |
|   +------+------+                                           |
|          |                                                  |
|          | timeout / drift                                   |
|          v                                                  |
|   +-------------+                                           |
|   |    STALE    |                                           |
|   +------+------+                                           |
|          |                                                  |
|          | re_calibrate()                                   |
|          +------------------------------------------------->|
|                      (to CALIBRATING)                       |
|                                                             |
+-------------------------------------------------------------+
```

---

## 8. Safety and Real-Time Standard

### 8.1 Safety-Critical Device Requirements

Devices marked `safety_critical: true` shall meet:

| Requirement | Implementation |
|-------------|----------------|
| Isolated Executor | Dedicated real-time thread on isolated core |
| Watchdog Timeout | 10 ms heartbeat; missed heartbeat triggers STOP |
| Redundant Path | Secondary bus/device monitored as hot standby |
| Deterministic Transport | Intra-process shared memory or EtherCAT DC only |
| Audit Logging | Tamper-resistant buffer for all commands and state changes |

### 8.2 Real-Time Contracts

| Condition | Latency | Jitter | Transport |
|-----------|---------|--------|-----------|
| Intra-process publish | < 1 us | < 100 ns | Shared memory |
| Intra-host publish | < 10 us | < 1 us | Shared memory |
| EtherCAT cycle (1 kHz) | < 10 us | < 1 us | EtherCAT DC |
| E-stop (T4) | < 1 ms | < 100 us | Shared memory |
| E-stop (T3) | < 10 ms | < 1 ms | Shared memory |

**Normative Requirement (SAF-001):** E-stop automatic recovery is prohibited. `clear_estop()` requires explicit authenticated command.

### 8.3 E-Stop Mechanism

```
+-------------------------------------------------------------+
|                    E-STOP MECHANISM                         |
|                                                             |
|   +-------------+                                           |
|   | E-Stop      |                                           |
|   | Trigger     |                                           |
|   | (HW or SW)  |                                           |
|   +------+------+                                           |
|          |                                                  |
|          | EMERGENCY_STOP event                             |
|          v                                                  |
|   +-------------+                                           |
|   | Event Bus   |                                           |
|   +------+------+                                           |
|          |                                                  |
|          | broadcast                                         |
|          v                                                  |
|   +-------------+                                           |
|   | All Actuator|                                           |
|   | Drivers     |                                           |
|   |             |                                           |
|   | emergency_stop()                                        |
|   |   - disable power                                       |
|   |   - estop_active = true                                 |
|   |   - publish state < 1 ms (T4)                            |
|   +-------------+                                           |
|                                                             |
|   Recovery: explicit authenticated clear_estop() only.      |
|                                                             |
+-------------------------------------------------------------+
```

---

## 9. Conformance and Validation

### 9.1 Conformance Levels

| Level | Description | Requirements |
|-------|-------------|--------------|
| **Level 1: Core** | Basic Spine compliance | Node lifecycle, HAL-API, message format |
| **Level 2: Standard** | Full driver compliance | Level 1 + diagnostics, calibration, config schema |
| **Level 3: Real-Time** | Deterministic behavior | Level 2 + RT scheduling, latency bounds |
| **Level 4: Safety** | Safety-critical certification | Level 3 + E-stop, audit, fault injection, watchdog |

### 9.2 Validation Test Matrix

| Test Category | T1 | T2 | T3 | T4 |
|---------------|----|----|----|----|
| Unit Tests | Required | Required | Required | Required |
| Integration Tests | Required | Required | Required | Required |
| Conformance Tests | Required | Required | Required | Required |
| Stress Tests (24h) | Optional | Required | Required | Required |
| Fault Injection | Optional | Required | Required | Required |
| Real-Time Tests | N/A | Optional | Required | Required |
| Safety Tests | N/A | N/A | Optional | Required |

**Normative Requirement (CONF-001):** A driver shall not be distributed as Spine-compliant unless it passes all required tests for its Complexity Tier.

---

## 10. Versioning and Evolution

### 10.1 Semantic Versioning

Spine uses **Semantic Versioning 2.0.0**:

```
MAJOR.MINOR.PATCH[-prerelease]
```

| Component | Meaning |
|-----------|---------|
| MAJOR | Breaking API change. All drivers must be updated. |
| MINOR | New feature, backward compatible. Drivers may opt-in. |
| PATCH | Bug fix, backward compatible. No driver changes required. |

### 10.2 API Compatibility

| API Version | Compatibility Guarantee |
|-------------|------------------------|
| `spine_hal_v1` | Supported for minimum 2 years after `spine_hal_v2` release. |
| `spine_core_v1` | Supported for minimum 3 years. |

### 10.3 Deprecation Policy

1. An API shall be marked deprecated in documentation and compiler warnings.
2. After deprecation, the API shall remain functional for one full minor release cycle.
3. After the grace period, the API may be removed in the next major release.

---

## 11. Publication Protocol

### 11.1 Document Identification

Every Spine technical document shall carry:

| Field | Format | Example |
|-------|--------|---------|
| Document ID | SPINE-{ABBREV}-{NNN} | SPINE-STD-001 |
| Version | MAJOR.MINOR.PATCH-status | 0.1.0-alpha |
| Date | ISO 8601 | 2026-07-31 |
| Status | Draft / Review / Approved / Superseded | Draft |

### 11.2 Change Log

```
+-------------------------------------------------------------+
|  CHANGE LOG                                                 |
|                                                             |
|  Version    Date        Author          Description         |
|  ---------  ----------  ------------    -------------------   |
|  0.1.0-a    2026-07-31  Spine Core    Initial draft         |
|                                                             |
+-------------------------------------------------------------+
```

### 11.3 Distribution

Spine technical standards shall be published via:

1. **GitHub Repository** — `github.com/ksaad20/Spine/docs/standards/`
2. **Zenodo Archive** — DOI-assigned, versioned releases
3. **Spine Website** — `spine-os.org/docs/`

### 11.4 Citation Format

```bibtex
@techreport{spine2026,
  title = {Spine OS Technical Standards v0.1},
  author = {{Spine Core Team}},
  year = {2026},
  month = {July},
  institution = {Spine Open Source Project},
  type = {Technical Standard},
  number = {SPINE-STD-001},
  url = {https://doi.org/10.5281/zenodo.xxxxx}
}
```

---

## 12. Appendices

### Appendix A: Normative CIDL Schema Summary

```cidl
package spine.standard;

// Core message header
message MessageHeader {
    magic: u32;        // 0x5350494E
    version: u16;
    msg_type: u16;
    reserved: u32;
    payload_size: u32;
    checksum: u32;
    timestamp_ns: u64;
    sequence: u32;
    flags: u32;
}

// Device descriptor
message DeviceDescriptor {
    id: string[128];
    class: DeviceClass;
    family: string[64];
    vendor: string[64];
    model: string[64];
    serial: string[64];
    firmware_version: string[32];
    hardware_version: string[32];
    bus: BusType;
    bus_address: string[128];
    capabilities: string[16][64];
    calibration_status: CalibrationState;
    health_status: HealthStatus;
    qos_profile: string[32];
}

enum DeviceClass {
    SENSOR = 0;
    ACTUATOR = 1;
    COMPUTE = 2;
    COMM = 3;
}

enum BusType {
    CAN = 0;
    CAN_FD = 1;
    ETHERCAT = 2;
    I2C = 3;
    SPI = 4;
    UART = 5;
    USB = 6;
    ETHERNET = 7;
    MIPI = 8;
    PCIE = 9;
}

enum CalibrationState {
    UNCALIBRATED = 0;
    CALIBRATING = 1;
    CALIBRATED = 2;
    STALE = 3;
}

enum HealthStatus {
    HEALTHY = 0;
    DEGRADED = 1;
    FAULT = 2;
    OFFLINE = 3;
}
```

### Appendix B: Driver Manifest Schema (Normative)

```yaml
# spine.driver.manifest.v1

driver:
  id: string                    # URI, required
  name: string                  # Required
  version: string               # SemVer, required
  api_version: string           # e.g., "spine_hal_v1", required
  description: string           # Optional
  author: string                # Optional
  license: string               # SPDX identifier, optional

  device:
    class: enum                 # SENSOR | ACTUATOR | COMPUTE | COMM
    family: string              # Required
    tier: enum                  # T1 | T2 | T3 | T4
    vendors: string[]           # Required
    models: string[]             # Required

  buses:
    - type: enum                # CAN | CAN_FD | ...
      required: bool
      modes: enum[]             # MASTER | SLAVE | CLIENT | MONITOR

  resources:
    cpu_affinity: int[]
    memory_mb: int
    memory_type: enum           # SHARED | DEVICE | PINNED
    realtime_priority: int      # 0-99
    deadline_us: int
    wcet_us: int

  ports:
    - name: string
      type: enum                # TOPIC | SERVICE | ACTION | EVENT
      direction: enum           # INPUT | OUTPUT | BIDIRECTIONAL
      message: string           # Fully qualified CIDL type
      qos: string                # QoS profile name
      rate_hz: int              # For streaming outputs
      max_size_bytes: int       # For bounded topics
      timeout_ms: int            # For services

  calibration:
    required: bool
    types: enum[]              # INTRINSIC | EXTRINSIC | TEMPORAL | DYNAMIC
    auto_calibrate: bool

  diagnostics:
    self_test: bool
    health_polling_hz: int
    temperature_monitoring: bool
    voltage_monitoring: bool

  safety:
    safety_critical: bool
    watchdog_timeout_ms: int
    recovery_policy: enum       # RESTART_NODE | RESTART_BUS | DEGRADE | STOP
    estop_response: enum        # NONE | DISABLE | BRAKE | FREE

  dependencies:
    libraries:
      - name: string
        version: string         # SemVer range
    kernel_modules: string[]
    system_packages: string[]
```

### Appendix C: Design Decision Records

#### ADR-STD-001: Why a Consolidated Technical Standard?

**Context:** Architecture, HAL, and driver specifications were developed as separate documents.  
**Decision:** Publish a single consolidated technical standard (this document) that normatively references the subordinate specifications.  
**Rationale:**
- A single standard is easier to cite, version, and audit.
- Subordinate specifications can evolve independently while the standard provides the stable contract.
- Certification bodies prefer a single reference document.

**Consequences:**
- The standard must be updated when subordinate specs change.
- Version coupling between standard and subordinate specs must be tracked.

#### ADR-STD-002: Why Apache-2.0 License?

**Context:** Robotics middleware licenses vary (BSD, MIT, GPL, proprietary).  
**Decision:** Spine technical standards and reference implementations are licensed under Apache-2.0.  
**Rationale:**
- Apache-2.0 is permissive and patent-protective.
- It is compatible with commercial robotics products.
- It is the standard license for major open-source infrastructure (Kubernetes, ROS 2).

**Consequences:**
- GPL-licensed components cannot be directly incorporated.
- Patent grants are explicit but narrow.

---

*End of Standard*
