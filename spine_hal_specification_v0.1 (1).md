# Spine OS — Hardware Abstraction Layer (HAL) Specification v0.1

**Document ID:** SPINE-HAL-001  
**Version:** 0.1.0-alpha  
**Date:** 2026-07-31  
**Status:** Draft — Phase 1 Foundation  
**Depends on:** SPINE-ARCH-001 (Platform Architecture)  
**Author:** Spine Core Team  

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [HAL Design Principles](#2-hal-design-principles)
3. [Device Taxonomy](#3-device-taxonomy)
4. [Driver Interface Contract](#4-driver-interface-contract)
5. [Device Lifecycle Model](#5-device-lifecycle-model)
6. [Configuration & Calibration Model](#6-configuration--calibration-model)
7. [HAL-to-Runtime Interface](#7-hal-to-runtime-interface)
8. [Error Handling & Diagnostics](#8-error-handling--diagnostics)
9. [Bus & Transport Abstraction](#9-bus--transport-abstraction)
10. [Safety & Real-Time Considerations](#10-safety--real-time-considerations)
11. [Reference Implementations](#11-reference-implementations)
12. [Glossary](#12-glossary)

---

## 1. Purpose & Scope

### 1.1 Mission Statement

The Hardware Abstraction Layer (HAL) is the **single boundary** between the Spine Runtime and the physical world. Its purpose is to:

- **Normalize** every sensor, actuator, compute accelerator, and communication peripheral into a **typed, portable interface**.
- **Isolate** the Runtime from vendor-specific protocols, register maps, timing quirks, and firmware versions.
- **Enable** hardware interchangeability — swapping a Velodyne for a Livox LiDAR, or a Dynamixel for a Harmonic Drive motor, must not require changes to application code.
- **Guarantee** deterministic behavior where the hardware permits it, and expose uncertainty explicitly where it does not.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| Driver interface specification | Application-level sensor fusion (lives in Runtime) |
| Device taxonomy and classification | Mechanical/electrical design of robot hardware |
| Bus abstraction (CAN, EtherCAT, I2C, SPI, UART, USB, Ethernet) | Low-level kernel drivers (provided by Linux/RTOS) |
| Configuration, calibration, and diagnostic APIs | AI model inference (lives in Compute Backend) |
| HAL-to-Runtime message contracts | Cloud connectivity protocols |
| Hot-plug and dynamic device discovery | Power management policy (lives in System Manager) |

### 1.3 HAL Position in the Stack

```
+-------------------------------------------------------------+
|                     APPLICATION LAYER                        |
|  +-------------+  +-------------+  +-------------+           |
|  | Perception  |  |  Planning   |  |   Control   |           |
|  |  Pipelines  |  |   Engines   |  |    Loops    |           |
|  +------+------+  +------+------+  +------+------+           |
+--------|----------|----------|----------|-------------------+
         |          |          |
         |          |          |
+--------|----------|----------|-------------------------------+
|                   SPINE RUNTIME LAYER                        |
|                                                              |
|  +--------------------------------------------------------+  |
|  |              COMPONENT FRAMEWORK (CFW)                 |  |
|  |   +--------+  +--------+  +--------+  +--------+       |  |
|  |   |  Node  |  |Service |  | Action |  | Event  |       |  |
|  |   |Manager |  | Router |  | Server |  |  Bus   |       |  |
|  |   +---+----+  +---+----+  +---+----+  +---+----+       |  |
|  +-------|----------|----------|----------|----------------+  |
|          |          |          |                            |
+----------|----------|----------|-----------------------------+
           |          |          |
           |          |          |
+----------|----------|----------|-----------------------------+
|              HAL-TO-RUNTIME INTERFACE                        |
|   +------------+  +------------+  +------------+             |
|   |   Device   |  |   Config   |  | Diagnostic |             |
|   |  Registry  |  |   Server   |  |    Hub     |             |
|   +------------+  +------------+  +------------+             |
+----------|----------|----------|-----------------------------+
           |          |          |
           |          |          |
+----------|----------|----------|-----------------------------+
|              HARDWARE ABSTRACTION LAYER (HAL)                |
|                                                              |
|  +--------------------------------------------------------+  |
|  |                  HAL CORE (HAL-CORE)                     |  |
|  |   +--------+  +--------+  +--------+  +--------+       |  |
|  |   | Device |  |  Bus   |  | Config |  | Safety |       |  |
|  |   |Manager |  | Router |  |Manager |  |Monitor |       |  |
|  |   +--------+  +--------+  +--------+  +--------+       |  |
|  +--------------------------------------------------------+  |
|                                                              |
|  +--------------------------------------------------------+  |
|  |               DRIVER FRAMEWORK (HAL-DFW)               |  |
|  |   +--------+  +--------+  +--------+  +--------+       |  |
|  |   | Sensor |  |Actuator|  |Compute |  |  Comm  |       |  |
|  |   | Driver |  | Driver |  | Driver |  | Driver |       |  |
|  |   +--------+  +--------+  +--------+  +--------+       |  |
|  +--------------------------------------------------------+  |
|                                                              |
|  +--------------------------------------------------------+  |
|  |                 BUS ADAPTERS (HAL-BUS)                 |  |
|  |   +------+ +------+ +------+ +------+ +------+         |  |
|  |   | CAN  | |Ether | | I2C  | | SPI  | | UART |         |  |
|  |   +------+ +------+ +------+ +------+ +------+         |  |
|  |   +------+ +------+ +------+ +------+                 |  |
|  |   | USB  | | Ether| | MIPI | | PCIe |                 |  |
|  |   +------+ +------+ +------+ +------+                 |  |
|  +--------------------------------------------------------+  |
+----------|----------|----------|-----------------------------+
           |          |          |
           |          |          |
+----------|----------|----------|-----------------------------+
|                      HARDWARE LAYER                          |
|   +------+  +------+  +------+  +------+  +------+          |
|   | LiDAR|  |Motor |  |Camera|  | IMU  |  |Network|          |
|   +------+  +------+  +------+  +------+  +------+          |
+-------------------------------------------------------------+
```

---

## 2. HAL Design Principles

### 2.1 Core Principles

| Principle | Rationale | Enforcement |
|-----------|-----------|-------------|
| **One Device, One Interface** | Every physical device exposes exactly one HAL interface, regardless of how many buses or protocols it uses internally. | The Device Manager enforces single-registration; a duplicate registration triggers a fault. |
| **Bus Agnosticism** | Application code must not know whether a motor is on CAN, EtherCAT, or UART. | Bus adapters are invisible to the Runtime; only the HAL Driver sees them. |
| **Fail-Fast Initialization** | A driver that cannot communicate with its hardware during `configure()` must fail immediately, not limp along. | The HAL Core validates hardware presence and protocol handshake before returning success. |
| **Calibration as First-Class** | Calibration is not an afterthought. Every device that requires it exposes a calibration state machine. | The Config Manager tracks calibration status per device; uncalibrated devices are marked `DEGRADED`. |
| **Diagnostics Are Mandatory** | Every driver must expose health, temperature, error counters, and firmware version. | The HAL-API rejects driver registration if the diagnostic interface is not implemented. |
| **No Hidden Blocking** | HAL operations must be non-blocking or explicitly timeout-bounded. The Runtime scheduler cannot be held hostage by a stuck I2C bus. | Bus adapters use async I/O with configurable timeouts; blocking calls are logged as violations. |

### 2.2 HAL Non-Goals

- **Replacing kernel drivers.** HAL sits above the kernel driver (e.g., the Linux `can` subsystem or `i2c-dev`). It does not implement bit-banged protocols.
- **Real-time guarantees on non-real-time buses.** If the underlying bus (e.g., USB) is inherently non-deterministic, HAL exposes this honestly in its QoS metadata.
- **Vendor-specific feature exposure.** If a device has a unique feature not covered by the standard taxonomy, it is exposed via an extension interface, not the primary API.

---

## 3. Device Taxonomy

### 3.1 Classification Hierarchy

```
DEVICE
|
+-- SENSOR
|   |
|   +-- RangeFinder
|   |   +-- LiDAR (1D, 2D, 3D)
|   |   +-- Ultrasonic
|   |   +-- Time-of-Flight (ToF)
|   |
|   +-- Camera
|   |   +-- RGB
|   |   +-- Depth (structured light, ToF, stereo)
|   |   +-- Thermal
|   |
|   +-- Inertial
|   |   +-- IMU (accelerometer + gyroscope)
|   |   +-- Magnetometer
|   |   +-- AHRS (attitude + heading)
|   |
|   +-- ForceTorque
|   +-- Encoder (rotary, linear)
|   +-- Tactile
|   +-- Gas / Chemical
|   +-- Temperature
|   +-- GPS / GNSS
|
+-- ACTUATOR
|   |
|   +-- Motor
|   |   +-- DC Brushed
|   |   +-- BLDC
|   |   +-- Stepper
|   |   +-- Servo (position-controlled)
|   |
|   +-- Gripper
|   |   +-- Parallel Jaw
|   |   +-- Vacuum
|   |   +-- Underactuated
|   |
|   +-- Linear Actuator
|   +-- Pneumatic Valve
|   +-- LED / Indicator
|
+-- COMPUTE
|   +-- GPU (CUDA, OpenCL, Vulkan)
|   +-- NPU / TPU (AI accelerator)
|   +-- FPGA
|   +-- DSP
|
+-- COMM
    +-- Ethernet Switch
    +-- WiFi Module
    +-- Bluetooth Module
    +-- Cellular (4G/5G)
    +-- LoRa / Sub-GHz
```

### 3.2 Device Metadata

Every device instance carries a **Device Descriptor**:

```yaml
device_descriptor:
  id: "spine://hal/lidar/front_3d"    # Unique URI in the deployment
  class: "SENSOR"
  family: "LiDAR"
  vendor: "Velodyne"
  model: "VLP-16"
  serial: "SN-12345678"
  firmware_version: "2.1.0"
  hardware_version: "Rev C"
  bus: "ETHERNET"
  bus_address: "192.168.1.201:2368"
  capabilities:
    - "3D_POINT_CLOUD"
    - "INTENSITY"
    - "RING_INDEX"
  calibration_status: "CALIBRATED"       # UNCALIBRATED | CALIBRATING | CALIBRATED | STALE
  health_status: "HEALTHY"             # HEALTHY | DEGRADED | FAULT | OFFLINE
  qos_profile: "SENSOR"                # References Spine Runtime QoS profile
```

### 3.3 Sensor Device Interface

All sensors expose the following base interface. Family-specific interfaces extend this.

```cidl
package spine.hal.sensor;

// Base interface for all sensor devices
service SensorBase {
    // Start streaming data
    method start_streaming(config: StreamingConfig) -> Result;

    // Stop streaming data
    method stop_streaming() -> Result;

    // Get latest reading (for polled sensors)
    method get_reading() -> SensorReading;

    // Get device health and diagnostics
    method get_diagnostics() -> SensorDiagnostics;

    // Trigger a self-test sequence
    method self_test() -> SelfTestResult;
}

message StreamingConfig {
    rate_hz: u16;           // Desired publish rate
    mode: StreamingMode;    // CONTINUOUS | TRIGGERED | ON_DEMAND

    enum StreamingMode {
        CONTINUOUS = 0;     // Publish at rate_hz
        TRIGGERED = 1;      // Publish on external trigger
        ON_DEMAND = 2;      // Publish only when get_reading() is called
    }
}

message SensorReading {
    timestamp_ns: u64;      // Acquisition time in unified Spine time
    frame_id: string[64];    // Coordinate frame of the sensor
    valid: bool;            // False if reading is known to be invalid

    // Family-specific payload is union-typed
    payload: oneof {
        point_cloud: PointCloud;
        image: Image;
        imu: ImuData;
        force_torque: ForceTorqueData;
        // ... etc
    }
}

message SensorDiagnostics {
    health: HealthStatus;
    temperature_c: f32;     // Device temperature, NaN if unavailable
    uptime_ms: u64;
    error_count: u32;
    warning_count: u32;
    last_error: string[256];

    enum HealthStatus {
        HEALTHY = 0;
        DEGRADED = 1;      // Still functional but suboptimal
        FAULT = 2;           // Not functional
        OFFLINE = 3;         // No communication
    }
}

message SelfTestResult {
    passed: bool;
    test_duration_ms: u32;
    details: string[512];
}
```

### 3.4 Actuator Device Interface

```cidl
package spine.hal.actuator;

service ActuatorBase {
    // Enable power to the actuator
    method enable() -> Result;

    // Disable power (safe state)
    method disable() -> Result;

    // Get current state
    method get_state() -> ActuatorState;

    // Get diagnostics
    method get_diagnostics() -> ActuatorDiagnostics;

    // Emergency stop (bypasses normal control)
    method emergency_stop() -> Result;

    // Clear emergency stop condition
    method clear_estop() -> Result;
}

message ActuatorState {
    timestamp_ns: u64;
    enabled: bool;
    estop_active: bool;

    // Present values
    position: f64;          // SI units (meters or radians)
    velocity: f64;          // SI units (m/s or rad/s)
    effort: f64;            // SI units (N or N·m)

    // Target values (if position/velocity/effort controlled)
    target_position: f64;
    target_velocity: f64;
    target_effort: f64;

    // Limits
    position_limit_min: f64;
    position_limit_max: f64;
    velocity_limit: f64;
    effort_limit: f64;

    // Temperature and power
    temperature_c: f32;
    bus_voltage_v: f32;
    bus_current_a: f32;
}

message ActuatorDiagnostics {
    health: HealthStatus;
    fault_flags: u32;       // Bitfield of active faults
    warning_flags: u32;     // Bitfield of active warnings
    temperature_c: f32;
    uptime_ms: u64;
    cycle_count: u64;        // Total motion cycles

    enum HealthStatus {
        HEALTHY = 0;
        DEGRADED = 1;
        FAULT = 2;
        OFFLINE = 3;
    }
}
```

### 3.5 Motor Extended Interface (Actuator Family)

Motors extend `ActuatorBase` with control mode support:

```cidl
package spine.hal.actuator.motor;

service MotorDriver extends spine.hal.actuator.ActuatorBase {
    // Set control mode
    method set_mode(mode: ControlMode) -> Result;

    // Send command (interpretation depends on mode)
    method command(cmd: MotorCommand) -> Result;

    // Set PID gains (if supported)
    method set_gains(gains: PidGains) -> Result;

    // Get PID gains
    method get_gains() -> PidGains;

    // Home / zero the motor
    method home() -> Result;
}

enum ControlMode {
    POSITION = 0;
    VELOCITY = 1;
    EFFORT = 2;
    POSITION_VELOCITY = 3;  // Cascaded: position outer, velocity inner
    IMPEDANCE = 4;           // Force-controlled with stiffness/damping
}

message MotorCommand {
    position: f64;           // Valid in POSITION, POSITION_VELOCITY, IMPEDANCE
    velocity: f64;           // Valid in VELOCITY, POSITION_VELOCITY
    effort: f64;             // Valid in EFFORT, IMPEDANCE
    stiffness: f64;         // Valid in IMPEDANCE (N/m or N·m/rad)
    damping: f64;             // Valid in IMPEDANCE
}

message PidGains {
    kp: f64;
    ki: f64;
    kd: f64;
    integral_limit: f64;
    output_limit: f64;
}
```

### 3.6 Compute Device Interface

```cidl
package spine.hal.compute;

service ComputeBackend {
    // Query device capabilities
    method get_capabilities() -> ComputeCapabilities;

    // Allocate memory on the device
    method allocate(size_bytes: u64, type: MemoryType) -> DeviceBuffer;

    // Deallocate memory
    method free(buffer: DeviceBuffer) -> Result;

    // Copy host -> device
    method upload(host_buffer: bytes, device_buffer: DeviceBuffer) -> Result;

    // Copy device -> host
    method download(device_buffer: DeviceBuffer) -> bytes;

    // Submit a compute job
    method submit(job: ComputeJob) -> JobHandle;

    // Wait for job completion
    method wait(handle: JobHandle, timeout_ms: u32) -> JobResult;

    // Get device utilization
    method get_utilization() -> ComputeUtilization;
}

message ComputeCapabilities {
    device_type: ComputeType;     // GPU | NPU | FPGA | DSP
    memory_total_bytes: u64;
    memory_bandwidth_gbps: f32;
    compute_tflops: f32;
    supports_fp16: bool;
    supports_fp32: bool;
    supports_int8: bool;
    supports_int4: bool;
    vendor_extensions: string[16][64];
}

enum ComputeType {
    GPU = 0;
    NPU = 1;
    FPGA = 2;
    DSP = 3;
}

message ComputeJob {
    job_type: JobType;            // INFERENCE | FFT | MATRIX | CUSTOM
    model_id: string[128];        // For inference jobs
    input_buffers: DeviceBuffer[8];
    output_buffers: DeviceBuffer[8];
    priority: u8;                 // 0 = highest
}

message ComputeUtilization {
    memory_used_bytes: u64;
    memory_total_bytes: u64;
    compute_utilization_pct: f32;  // 0.0 - 100.0
    temperature_c: f32;
    power_draw_w: f32;
}
```

---

## 4. Driver Interface Contract

### 4.1 Driver as a Spine Node

Every HAL driver is a **Spine Node** (see SPINE-ARCH-001, Section 3.1.1). It implements:

- The **Node Lifecycle State Machine** (`configure`, `activate`, `deactivate`, `cleanup`).
- **Input ports** for commands from the Runtime.
- **Output ports** for sensor data, diagnostics, and events.

```
+-------------------------------------------------------------+
|                    HAL DRIVER NODE                           |
|                                                              |
|  +--------------------------------------------------------+  |
|  |              NODE STATE MACHINE                          |  |
|  |                                                          |  |
|  |   UNCONFIGURED                                           |  |
|  |      |                                                   |  |
|  |      | configure()                                       |  |
|  |      v                                                   |  |
|  |   INACTIVE                                               |  |
|  |      |                                                   |  |
|  |      | activate()                                        |  |
|  |      v                                                   |  |
|  |   ACTIVE                                                 |  |
|  |      |                                                   |  |
|  |      | deactivate()                                      |  |
|  |      v                                                   |  |
|  |   INACTIVE                                               |  |
|  |      |                                                   |  |
|  |      | cleanup()                                         |  |
|  |      v                                                   |  |
|  |   FINALIZED                                              |  |
|  |                                                          |  |
|  +--------------------------------------------------------+  |
|                                                              |
|  INPUT PORTS              OUTPUT PORTS                       |
|  +----------------+       +----------------+                |
|  | /cmd           |       | /data          |                |
|  | (ActuatorCmd)  |       | (SensorReading)|                |
|  +----------------+       +----------------+                |
|  | /config        |       | /diagnostics   |                |
|  | (ConfigReq)    |       | (Health)       |                |
|  +----------------+       +----------------+                |
|  | /calibrate     |       | /events        |                |
|  | (CalCmd)       |       | (DeviceEvent)  |                |
|  +----------------+       +----------------+                |
|                                                              |
|  +--------------------------------------------------------+  |
|  |              BUS ADAPTER INTERFACE                       |  |
|  |                                                          |  |
|  |   read() | write() | ioctl() | flush() | reset()        |  |
|  |                                                          |  |
|  +--------------------------------------------------------+  |
|                                                              |
+-------------------------------------------------------------+
```

### 4.2 Driver Registration Contract

When a driver is loaded, it must register with the HAL Core by providing a **Driver Manifest**:

```yaml
driver_manifest:
  name: "velodyne_vlp16_driver"
  version: "1.2.0"
  api_version: "spine_hal_v1"

  device_class: "SENSOR"
  device_family: "LiDAR"
  supported_vendors: ["Velodyne"]
  supported_models: ["VLP-16", "VLP-32C", "VLS-128"]

  required_buses: ["ETHERNET"]

  resource_budget:
    cpu_cores: [0]
    memory_mb: 64
    realtime_priority: 40

  ports:
    - name: "/velodyne_front/points"
      type: "TOPIC"
      direction: "OUTPUT"
      message_type: "spine.sensor.PointCloud"
      qos: "SENSOR"

    - name: "/velodyne_front/cmd"
      type: "SERVICE"
      direction: "INPUT"
      message_type: "spine.hal.sensor.SensorBase"

    - name: "/velodyne_front/diagnostics"
      type: "TOPIC"
      direction: "OUTPUT"
      message_type: "spine.hal.sensor.SensorDiagnostics"
      qos: "TELEMETRY"
      rate_hz: 1

  calibration:
    required: false
    type: "EXTRINSIC"           # INTRINSIC | EXTRINSIC | BOTH

  diagnostics:
    self_test_supported: true
    health_polling_hz: 10
    temperature_monitoring: true
```

### 4.3 Driver Implementation Requirements

| Requirement | Severity | Description |
|-------------|----------|-------------|
| **Thread Safety** | Mandatory | Drivers must be thread-safe if the Bus Adapter uses multiple threads. |
| **Timeout Handling** | Mandatory | Every bus operation must have a configurable timeout. Default: 100 ms. |
| **Zero-Copy Output** | Mandatory | Sensor data must be published via the zero-copy shared memory path when intra-host. |
| **Timestamp Fidelity** | Mandatory | `timestamp_ns` must reflect the moment of physical acquisition, not the moment of publication. |
| **Graceful Degradation** | Mandatory | On partial failure (e.g., dropped packets), the driver must publish a `valid=false` reading, not crash. |
| **Hot-Plug Support** | Recommended | Drivers should detect device disconnection and reconnection without process restart. |
| **Firmware Update** | Optional | Drivers may expose a firmware update interface via the Config Server. |

---

## 5. Device Lifecycle Model

### 5.1 HAL-Specific Lifecycle States

HAL extends the Spine Node lifecycle with device-specific substates:

```
+-------------------------------------------------------------+
|                     DEVICE LIFECYCLE                         |
|                                                              |
|   UNCONFIGURED                                               |
|       |                                                      |
|       | configure()                                          |
|       v                                                      |
|   +----------+                                               |
|   | INACTIVE |                                               |
|   +----+-----+                                               |
|        |                                                     |
|        | activate()                                          |
|        v                                                     |
|   +----------+                                               |
|   |  ACTIVE  |                                               |
|   +----+-----+                                               |
|        |                                                     |
|   +----+----+----+----+                                    |
|   |         |         |         |                            |
|   v         v         v         v                            |
| STREAMING CALIBRATING DEGRADED  FAULT                        |
|   |         |         |         |                            |
|   |         |         |         |                            |
|   +----+----+----+----+                                    |
|        |                                                     |
|        | deactivate()                                       |
|        v                                                     |
|   +----------+                                               |
|   | INACTIVE |                                               |
|   +----+-----+                                               |
|        |                                                     |
|        | cleanup()                                           |
|        v                                                     |
|   +----------+                                               |
|   | FINALIZED|                                               |
|   +----------+                                               |
|                                                              |
+-------------------------------------------------------------+
```

| Substate | Description | Trigger |
|----------|-------------|---------|
| **STREAMING** | Device is actively producing data. | `start_streaming()` succeeds. |
| **CALIBRATING** | Device is running a calibration routine. | `calibrate()` called or auto-calibration triggered. |
| **DEGRADED** | Device is functional but operating outside spec (e.g., high temp, reduced accuracy). | Health monitor detects threshold breach. |
| **FAULT** | Device has encountered an unrecoverable error. | Hardware fault, bus error, or self-test failure. |

### 5.2 State Transitions and Runtime Interaction

| From | To | Condition | Runtime Action |
|------|-----|-----------|----------------|
| INACTIVE | ACTIVE | `activate()` succeeds, hardware responds. | Node enters Active; HAL publishes `DeviceOnline` event. |
| ACTIVE | STREAMING | `start_streaming()` succeeds. | Topic begins publishing at configured rate. |
| ACTIVE | CALIBRATING | Calibration initiated. | Data publishing paused; calibration progress published on `/events`. |
| CALIBRATING | STREAMING | Calibration succeeds. | Calibration parameters saved; data publishing resumes. |
| CALIBRATING | DEGRADED | Calibration fails but device still functional. | Runtime notified; application may retry or use degraded mode. |
| ACTIVE | DEGRADED | Health threshold exceeded. | `DeviceDegraded` event published; data continues with `valid` flag logic. |
| DEGRADED | FAULT | Health continues to degrade. | `DeviceFault` event published; driver attempts graceful shutdown. |
| ANY | OFFLINE | Bus communication lost. | `DeviceOffline` event published; HAL retries reconnection. |
| OFFLINE | INACTIVE | Device reconnected and re-initialized. | `DeviceOnline` event published; application may re-activate. |

---

## 6. Configuration & Calibration Model

### 6.1 Configuration Hierarchy

Spine HAL uses a **three-tier configuration model**:

```
+-------------------------------------------------------------+
|  TIER 1: VENDOR DEFAULTS                                     |
|  Shipped with the driver.                                   |
|  Read-only.                                                   |
+-------------------------------------------------------------+
              |
              | override
              v
+-------------------------------------------------------------+
|  TIER 2: DEPLOYMENT CONFIG                                   |
|  Loaded from System Manifest.                                 |
|  Overrides vendor defaults.                                   |
+-------------------------------------------------------------+
              |
              | override
              v
+-------------------------------------------------------------+
|  TIER 3: RUNTIME PARAMETERS                                  |
|  Set via Parameter Server at runtime.                         |
|  Overrides deployment config.                                   |
|  Not persisted unless explicitly saved.                         |
+-------------------------------------------------------------+
```

### 6.2 Configuration Schema

Every driver declares its configuration schema in the Driver Manifest:

```yaml
config_schema:
  - key: "frame_id"
    type: "string"
    default: "lidar_front"
    description: "TF frame ID for this device"
    mutable: false          # Cannot change at runtime

  - key: "ip_address"
    type: "string"
    default: "192.168.1.201"
    description: "Device IP address"
    mutable: false

  - key: "publish_rate_hz"
    type: "u16"
    default: 10
    min: 1
    max: 30
    description: "Point cloud publish rate"
    mutable: true             # Can change at runtime

  - key: "range_min_m"
    type: "f64"
    default: 0.4
    description: "Minimum valid range"
    mutable: true

  - key: "calibration.extrinsic.transform"
    type: "f64[16]"           # 4x4 row-major homogeneous transform
    default: "identity"
    description: "Extrinsic calibration: sensor -> base_link"
    mutable: true
    requires_calibration: true
```

### 6.3 Calibration Framework

Calibration is a **first-class operation** in HAL, not an afterthought.

#### 6.3.1 Calibration Types

| Type | Description | Persistence |
|------|-------------|-------------|
| **Intrinsic** | Internal device parameters (camera matrix, LiDAR beam angles, IMU scale factors). | Stored in device firmware or driver persistent storage. |
| **Extrinsic** | Spatial relationship between device and robot frame (6-DOF transform). | Stored in the Spine Parameter Server; versioned per deployment. |
| **Temporal** | Time offset between device clock and Spine unified time domain. | Computed at runtime via PTP or manual offset; not persisted. |
| **Dynamic** | Parameters that change during operation (temperature compensation, drift correction). | Updated continuously by driver; may be persisted as statistics. |

#### 6.3.2 Calibration State Machine

```
+-------------------------------------------------------------+
|                  CALIBRATION STATE MACHINE                   |
|                                                              |
|   +-------------+                                            |
|   | UNCALIBRATED|                                            |
|   +------+------+                                            |
|          |                                                    |
|          | start_calibration()                                |
|          v                                                    |
|   +-------------+                                            |
|   | CALIBRATING |                                            |
|   +------+------+                                            |
|          |                                                    |
|          | success                                            |
|          v                                                    |
|   +-------------+                                            |
|   |  CALIBRATED |                                            |
|   +------+------+                                            |
|          |                                                    |
|          | timeout / drift                                     |
|          v                                                    |
|   +-------------+                                            |
|   |    STALE    |                                            |
|   +------+------+                                            |
|          |                                                    |
|          | re_calibrate()                                     |
|          +--------------------------------------------------->|
|                         (back to CALIBRATING)                 |
|                                                              |
+-------------------------------------------------------------+
```

#### 6.3.3 Calibration API

```cidl
package spine.hal.calibration;

service CalibrationManager {
    // Start a calibration routine for a specific device
    method calibrate_device(device_id: string, type: CalibrationType) -> CalibrationJob;

    // Get calibration status for a device
    method get_status(device_id: string) -> CalibrationStatus;

    // Load calibration from persistent storage
    method load_calibration(device_id: string, source: CalibrationSource) -> Result;

    // Save calibration to persistent storage
    method save_calibration(device_id: string, target: CalibrationSource) -> Result;

    // Get calibration parameters
    method get_parameters(device_id: string) -> CalibrationParameters;

    // Set calibration parameters (for manual / external calibration)
    method set_parameters(device_id: string, params: CalibrationParameters) -> Result;
}

message CalibrationJob {
    job_id: u64;
    device_id: string[128];
    type: CalibrationType;
    status: JobStatus;       # PENDING | RUNNING | SUCCESS | FAILED | CANCELLED
    progress_pct: f32;
    started_ns: u64;
    estimated_completion_ns: u64;
}

enum CalibrationType {
    INTRINSIC = 0;
    EXTRINSIC = 1;
    TEMPORAL = 2;
    DYNAMIC = 3;
}

message CalibrationStatus {
    device_id: string[128];
    state: CalibrationState;
    last_calibrated_ns: u64;
    confidence: f32;           # 0.0 - 1.0
    drift_estimate: Transform; # Estimated drift since last calibration
}

enum CalibrationState {
    UNCALIBRATED = 0;
    CALIBRATING = 1;
    CALIBRATED = 2;
    STALE = 3;
}
```

---

## 7. HAL-to-Runtime Interface

### 7.1 HAL-API Services

The HAL exposes four services to the Spine Runtime:

```
+-------------------------------------------------------------+
|                     SPINE RUNTIME                            |
|                                                              |
|  +--------------------------------------------------------+  |
|  |                   HAL-API SERVICES                       |  |
|  |                                                          |  |
|  |  +-----------------+  +-----------------+              |  |
|  |  |  Device Registry  |  |  Config Server   |              |  |
|  |  |                 |  |                 |              |  |
|  |  |  - register()   |  |  - get()        |              |  |
|  |  |  - unregister() |  |  - set()        |              |  |
|  |  |  - list()       |  |  - save()       |              |  |
|  |  |  - lookup()     |  |  - load()       |              |  |
|  |  +-----------------+  +-----------------+              |  |
|  |                                                          |  |
|  |  +-----------------+  +-----------------+              |  |
|  |  |  Diagnostic Hub |  |  Event Notifier  |              |  |
|  |  |                 |  |                 |              |  |
|  |  |  - subscribe()  |  |  - publish()    |              |  |
|  |  |  - query()      |  |  - subscribe()  |              |  |
|  |  |  - export()     |  |                 |              |  |
|  |  |  - clear()      |  |                 |              |  |
|  |  +-----------------+  +-----------------+              |  |
|  |                                                          |  |
|  +--------------------------------------------------------+  |
|                                                              |
+-------------------------------------------------------------+
```

### 7.2 Device Registry

```cidl
package spine.hal.api;

service DeviceRegistry {
    // Register a new device instance
    method register(descriptor: DeviceDescriptor) -> RegistrationResult;

    // Unregister a device (e.g., on hot-unplug)
    method unregister(device_id: string) -> Result;

    // List all registered devices, optionally filtered
    method list(filter: DeviceFilter) -> DeviceDescriptor[];

    // Lookup a device by ID
    method lookup(device_id: string) -> DeviceDescriptor;

    // Check if a device ID is available
    method is_available(device_id: string) -> bool;
}

message RegistrationResult {
    success: bool;
    assigned_id: string[128];    # May differ from requested if collision
    error: string[256];
}

message DeviceFilter {
    class_filter: DeviceClass;     # ANY | SENSOR | ACTUATOR | COMPUTE | COMM
    family_filter: string[64];
    health_filter: HealthStatus;   # ANY | HEALTHY | DEGRADED | FAULT | OFFLINE
    bus_filter: BusType;           # ANY | CAN | ETHERCAT | I2C | SPI | UART | ETHERNET | USB
}
```

### 7.3 Config Server

```cidl
service ConfigServer {
    // Get a configuration value
    method get(device_id: string, key: string) -> ConfigValue;

    // Set a configuration value (runtime mutable only)
    method set(device_id: string, key: string, value: ConfigValue) -> Result;

    // Get all configuration for a device
    method get_all(device_id: string) -> ConfigEntry[];

    // Save current runtime config to persistent storage
    method save(device_id: string, target: ConfigTarget) -> Result;

    // Load config from persistent storage
    method load(device_id: string, source: ConfigSource) -> Result;

    // Reset to defaults
    method reset(device_id: string) -> Result;
}

message ConfigValue {
    type: ConfigType;              # BOOL | I8 | I16 | I32 | I64 | U8 | U16 | U32 | U64 | F32 | F64 | STRING | BYTES | ARRAY
    bool_value: bool;
    int_value: i64;
    float_value: f64;
    string_value: string[1024];
    bytes_value: bytes[4096];
    array_value: ConfigValue[64];
}

enum ConfigTarget {
    DEPLOYMENT_MANIFEST = 0;       # Update the manifest file
    PARAMETER_SERVER = 1;          # Update the Runtime Parameter Server
    DEVICE_FIRMWARE = 2;           # Write to device flash (if supported)
}
```

### 7.4 Diagnostic Hub

```cidl
service DiagnosticHub {
    // Subscribe to diagnostic stream for a device or all devices
    method subscribe(filter: DiagnosticFilter) -> DiagnosticStream;

    // Query historical diagnostics
    method query(device_id: string, start_ns: u64, end_ns: u64) -> DiagnosticRecord[];

    // Export diagnostics to a file
    method export(device_id: string, format: ExportFormat, path: string) -> Result;

    // Clear diagnostic history for a device
    method clear(device_id: string) -> Result;
}

message DiagnosticFilter {
    device_id: string[128];         # "" means all devices
    severity: SeverityFilter;        # DEBUG | INFO | WARN | ERROR | FATAL | ALL
    category: string[64];           # e.g., "temperature", "bus_error", "timing"
}

message DiagnosticRecord {
    timestamp_ns: u64;
    device_id: string[128];
    severity: Severity;
    category: string[64];
    code: u32;
    message: string[512];
    data: bytes[1024];             # Optional binary diagnostic payload
}

enum Severity {
    DEBUG = 0;
    INFO = 1;
    WARN = 2;
    ERROR = 3;
    FATAL = 4;
}
```

### 7.5 Event Notifier

```cidl
service EventNotifier {
    // Publish a device event to the system event bus
    method publish(event: DeviceEvent) -> Result;

    // Subscribe to device events with filtering
    method subscribe(filter: EventFilter) -> EventStream;
}

message DeviceEvent {
    timestamp_ns: u64;
    device_id: string[128];
    event_type: EventType;
    previous_state: DeviceState;
    current_state: DeviceState;
    details: string[512];
}

enum EventType {
    DEVICE_REGISTERED = 0;
    DEVICE_ONLINE = 1;
    DEVICE_OFFLINE = 2;
    DEVICE_DEGRADED = 3;
    DEVICE_FAULT = 4;
    CALIBRATION_STARTED = 5;
    CALIBRATION_COMPLETED = 6;
    CALIBRATION_FAILED = 7;
    CONFIG_CHANGED = 8;
    FIRMWARE_UPDATED = 9;
    EMERGENCY_STOP = 10;
    EMERGENCY_CLEARED = 11;
}

enum DeviceState {
    UNCONFIGURED = 0;
    INACTIVE = 1;
    ACTIVE = 2;
    STREAMING = 3;
    CALIBRATING = 4;
    DEGRADED = 5;
    FAULT = 6;
    OFFLINE = 7;
    FINALIZED = 8;
}
```

---

## 8. Error Handling & Diagnostics

### 8.1 Error Taxonomy

HAL errors are classified into four categories:

| Category | Description | Examples |
|----------|-------------|----------|
| **Bus Errors** | Communication layer failures. | Timeout, CRC error, bus-off (CAN), framing error. |
| **Device Errors** | Hardware or firmware failures. | Over-temperature, self-test failure, firmware crash. |
| **Configuration Errors** | Invalid or inconsistent configuration. | Missing required parameter, out-of-range value, calibration mismatch. |
| **Resource Errors** | System resource exhaustion. | Out of memory, CPU starvation, file descriptor limit. |

### 8.2 Error Codes

All HAL errors use a structured error code:

```
+-------------------------------------------------------------+
|              HAL ERROR CODE (32-bit)                        |
|                                                              |
|   +--------+--------+--------+--------+                    |
|   |Category|  Bus   | Device |  Code  |                    |
|   |  4b    |  4b    |  8b    |  16b   |                    |
|   +--------+--------+--------+--------+                    |
|                                                              |
+-------------------------------------------------------------+
```

| Category Code | Meaning |
|---------------|---------|
| `0x0` | Success |
| `0x1` | Bus Error |
| `0x2` | Device Error |
| `0x3` | Configuration Error |
| `0x4` | Resource Error |
| `0x5` | Security Error |
| `0xF` | Unknown / Internal |

Example: `0x12000005` = Bus Error (0x1), CAN bus (0x2), Generic timeout (0x0005).

### 8.3 Fault Injection and Recovery

HAL supports **programmable fault injection** for testing:

```yaml
fault_injection:
  enabled: true                  # Only in debug / test builds

  rules:
    - device_id: "lidar_front"
      trigger: "random"
      probability: 0.001           # 0.1% chance per packet
      fault_type: "DROP_PACKET"
      duration_ms: 100

    - device_id: "motor_left"
      trigger: "time"
      after_ns: 60000000000      # 60 seconds after activation
      fault_type: "BUS_OFF"
      duration_ms: 500
```

### 8.4 Watchdog Integration

Every HAL driver is monitored by a **HAL Watchdog**:

- **Heartbeat:** Drivers must send a heartbeat message every `heartbeat_period_ms` (default: 100 ms).
- **Missed Heartbeats:** After 3 missed heartbeats, the HAL Core marks the device `OFFLINE` and attempts restart.
- **Recovery Policy:** Configurable per device:
  - `RESTART_NODE` — Restart the driver process.
  - `RESTART_BUS` — Reset the bus adapter.
  - `DEGRADE` — Continue with last-known-good data.
  - `STOP` — Halt the robot (safety-critical devices only).

---

## 9. Bus & Transport Abstraction

### 9.1 Bus Adapter Interface

Every bus type in HAL implements a common **Bus Adapter** interface:

```cidl
package spine.hal.bus;

service BusAdapter {
    // Open the bus with configuration
    method open(config: BusConfig) -> Result;

    // Close the bus
    method close() -> Result;

    // Read data from a device on the bus
    method read(address: BusAddress, length: u32, timeout_ms: u32) -> bytes;

    // Write data to a device on the bus
    method write(address: BusAddress, data: bytes, timeout_ms: u32) -> Result;

    // Perform a read-modify-write or custom transaction
    method transaction(txn: BusTransaction) -> BusTransactionResult;

    // Flush pending writes
    method flush() -> Result;

    // Reset the bus (e.g., CAN bus-off recovery)
    method reset() -> Result;

    // Get bus health and statistics
    method get_status() -> BusStatus;
}

message BusConfig {
    bus_type: BusType;
    device_path: string[128];      # e.g., "/dev/can0", "192.168.1.1"
    bitrate: u32;                  # e.g., 1000000 for 1 Mbps CAN
    mode: BusMode;                 # MASTER | SLAVE | MONITOR

    # Bus-specific parameters
    parameters: ConfigEntry[];
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

message BusStatus {
    state: BusState;               # OPEN | CLOSED | ERROR | RECOVERING
    error_count: u32;
    tx_count: u64;
    rx_count: u64;
    tx_bytes: u64;
    rx_bytes: u64;
    bus_load_pct: f32;            # 0.0 - 100.0
    last_error: string[256];
}
```

### 9.2 Bus-Specific Considerations

| Bus | Spine HAL Abstraction | Key Considerations |
|-----|----------------------|-------------------|
| **CAN / CAN-FD** | SocketCAN integration. Frame filtering at kernel level. Bus-off recovery handled by HAL. | Deterministic up to bus load ~70%. Beyond that, frames may be dropped. |
| **EtherCAT** | SOEM or custom master. Cyclic data exchange with DC (Distributed Clocks) synchronization. | Sub-microsecond jitter possible with DC. Requires real-time kernel. |
| **I2C** | `i2c-dev` userspace. Clock stretching support. Multi-master detection. | Not deterministic. Timeout-bounded only. Avoid for safety-critical paths. |
| **SPI** | `spidev` userspace. Configurable clock, mode, bits. Chip select management. | Full-duplex, high throughput. Chip select may require GPIO if not hardware-managed. |
| **UART** | `termios` or custom. Configurable baud, parity, flow control. | Simple but slow. Good for debug, legacy devices, or MCU bridges. |
| **Ethernet** | Raw sockets or UDP. Used for high-bandwidth devices (LiDAR, cameras). | Best-effort. Use eUDP (Spine enhanced UDP) for reliability without TCP overhead. |
| **USB** | `libusb` or kernel drivers. Hot-plug support via udev. | Non-deterministic due to host controller scheduling. Good for cameras, not for control. |
| **MIPI CSI-2** | V4L2 or vendor SDK. Used for camera pipelines. | High bandwidth, low latency. Requires kernel driver support. |

---

## 10. Safety & Real-Time Considerations

### 10.1 Safety-Critical Device Requirements

Devices marked `safety_critical: true` in their manifest receive special treatment:

| Requirement | Implementation |
|-------------|----------------|
| **Isolated Executor** | Runs on a dedicated real-time thread, isolated from best-effort Nodes. |
| **Watchdog Timeout** | Heartbeat period reduced to 10 ms; missed heartbeat triggers `STOP` recovery. |
| **Redundant Path** | If available, a secondary bus or device is monitored as a hot standby. |
| **Deterministic Transport** | Only intra-process shared memory or EtherCAT DC is permitted. No UDP, no USB. |
| **Audit Logging** | Every command and state change is logged to a tamper-resistant buffer. |

### 10.2 Real-Time Guarantees

HAL makes the following real-time contracts:

| Condition | Guarantee |
|-----------|-----------|
| Intra-process sensor publish | < 1 us latency, zero-copy. |
| Inter-process sensor publish (same host) | < 10 us latency, shared memory. |
| EtherCAT cycle (1 kHz) | < 10 us jitter with DC sync and `SCHED_FIFO`. |
| CAN-FD frame (64 bytes @ 2 Mbps) | < 500 us transmission time. |
| I2C read (8 bytes @ 400 kHz) | < 500 us, but not deterministic due to clock stretching. |
| USB UVC frame (1080p) | ~16 ms frame time, non-deterministic start. |

HAL **does not** guarantee real-time behavior on:
- WiFi or Bluetooth transports.
- USB buses under heavy load.
- Non-real-time Linux kernels without `PREEMPT_RT`.
- Virtualized or containerized environments without CPU pinning.

### 10.3 Emergency Stop (E-Stop)

HAL provides a **system-level E-Stop mechanism**:

```
+-------------------------------------------------------------+
|                    E-STOP MECHANISM                          |
|                                                              |
|   +-------------+                                            |
|   |  E-Stop     |                                            |
|   |  Trigger    |                                            |
|   |  (HW/SW)    |                                            |
|   +------+------+                                            |
|          |                                                    |
|          | publish EMERGENCY_STOP event                      |
|          v                                                    |
|   +-------------+                                            |
|   | Event Bus   |                                            |
|   +------+------+                                            |
|          |                                                    |
|          | broadcast to all actuators                       |
|          v                                                    |
|   +-------------+                                            |
|   | Actuator    |                                            |
|   | Drivers     |                                            |
|   |             |                                            |
|   |  emergency_stop()                                       |  |
|   |   - disable power                                       |  |
|   |   - set estop_active = true                             |  |
|   |   - publish state within 1 ms                           |  |
|   +-------------+                                            |
|                                                              |
|   Recovery requires explicit, authenticated command.         |
|   Automatic recovery is PROHIBITED.                         |
|                                                              |
+-------------------------------------------------------------+
```

---

## 11. Reference Implementations

### 11.1 Minimum Viable Driver Template

```cpp
// spine/hal/drivers/template_driver.hpp
#pragma once
#include <spine/hal/driver_base.hpp>
#include <spine/hal/sensor/sensor_base.hpp>

namespace spine::hal::drivers {

class TemplateDriver : public DriverBase, public sensor::SensorBase {
public:
    // Node lifecycle
    Result on_configure(const NodeConfig& config) override;
    Result on_activate() override;
    Result on_deactivate() override;
    Result on_cleanup() override;

    // SensorBase interface
    Result start_streaming(const StreamingConfig& config) override;
    Result stop_streaming() override;
    SensorReading get_reading() override;
    SensorDiagnostics get_diagnostics() override;
    SelfTestResult self_test() override;

    // Driver-specific
    Result open_bus(const BusConfig& bus_config);
    Result close_bus();

private:
    std::unique_ptr<BusAdapter> bus_;
    DeviceDescriptor descriptor_;
    bool streaming_ = false;
    StreamingConfig stream_config_;
    std::thread stream_thread_;

    void streaming_loop();
};

} // namespace spine::hal::drivers
```

### 11.2 Driver Manifest (YAML)

```yaml
# velodyne_vlp16.manifest.yaml
driver:
  name: "velodyne_vlp16_driver"
  version: "1.2.0"
  api_version: "spine_hal_v1"

  device:
    class: "SENSOR"
    family: "LiDAR"
    vendors: ["Velodyne"]
    models: ["VLP-16", "VLP-32C"]

  buses:
    - type: "ETHERNET"
      required: true

  resources:
    cpu_affinity: [2, 3]
    memory_mb: 128
    realtime_priority: 40

  ports:
    - name: "/velodyne/points"
      type: "TOPIC"
      direction: "OUTPUT"
      message: "spine.sensor.PointCloud"
      qos: "SENSOR"

    - name: "/velodyne/diagnostics"
      type: "TOPIC"
      direction: "OUTPUT"
      message: "spine.hal.sensor.SensorDiagnostics"
      qos: "TELEMETRY"
      rate_hz: 1

    - name: "/velodyne/cmd"
      type: "SERVICE"
      direction: "INPUT"
      message: "spine.hal.sensor.SensorBase"

  calibration:
    required: false
    types: ["EXTRINSIC"]

  diagnostics:
    self_test: true
    health_polling_hz: 10
    temperature: true

  safety:
    safety_critical: false
    watchdog_timeout_ms: 100
    recovery_policy: "RESTART_NODE"
```

---

## 12. Glossary

| Term | Definition |
|------|-----------|
| **Bus Adapter** | The HAL component that abstracts a specific physical bus (CAN, I2C, SPI, etc.) into the generic `BusAdapter` interface. |
| **Calibration** | The process of determining intrinsic, extrinsic, temporal, or dynamic parameters that map raw device output to physically meaningful data. |
| **Device Class** | The top-level category of a device: SENSOR, ACTUATOR, COMPUTE, or COMM. |
| **Device Family** | The sub-category within a class, e.g., LiDAR under SENSOR, or Motor under ACTUATOR. |
| **Driver Manifest** | A declarative YAML file that describes a driver's capabilities, requirements, ports, and metadata. |
| **E-Stop** | Emergency Stop. A system-level safety mechanism that immediately disables all actuators. |
| **HAL Core** | The central HAL component that manages the Device Registry, Config Server, Diagnostic Hub, and Event Notifier. |
| **HAL-API** | The interface boundary between the HAL and the Spine Runtime. |
| **Hot-Plug** | The ability to add or remove hardware devices at runtime without restarting the system. |
| **Sensor Reading** | The standardized message format that all sensors publish, containing a timestamp, frame ID, validity flag, and family-specific payload. |
| **Zero-Copy** | A transport optimization where message data is shared between publisher and subscriber via shared memory without serialization or memory copying. |

---

## Appendix A: Design Decision Records

### ADR-HAL-001: Why a Unified Bus Adapter Interface?

**Context:** Different buses (CAN, I2C, SPI, Ethernet) have fundamentally different semantics.  
**Decision:** HAL defines a single `BusAdapter` interface that all bus types implement, rather than exposing bus-specific APIs to drivers.  
**Rationale:**
- Drivers should not care whether a motor is on CAN or EtherCAT.
- Enables transparent bus migration (e.g., moving a sensor from I2C to SPI without driver changes).
- Simplifies testing — bus adapters can be mocked uniformly.

**Consequences:**
- Some bus-specific features (e.g., EtherCAT Distributed Clocks) are exposed via the generic `transaction()` method with bus-specific parameter structs.
- Bus adapters must handle protocol complexity internally.

### ADR-HAL-002: Why Calibration as a First-Class HAL Feature?

**Context:** In many robotics stacks, calibration is handled ad-hoc by application nodes.  
**Decision:** HAL owns the calibration state machine, persistence, and API for all devices.  
**Rationale:**
- Calibration is a property of the hardware, not the application.
- Centralizing calibration prevents drift from inconsistent application-level implementations.
- Enables system-wide calibration validation at startup.

**Consequences:**
- HAL drivers must implement calibration callbacks.
- The Parameter Server must support large binary calibration blobs (e.g., camera distortion maps).

### ADR-HAL-003: Why Mandatory Diagnostics for Every Driver?

**Context:** Debugging hardware issues in distributed robotics systems is notoriously difficult.  
**Decision:** Every HAL driver must expose health, temperature, error counts, and self-test capability.  
**Rationale:**
- Without uniform diagnostics, operators cannot distinguish software bugs from hardware faults.
- Health monitoring enables predictive maintenance and automatic degradation.
- Self-tests catch latent faults before they cause mission failure.

**Consequences:**
- Driver development overhead increases.
- HAL Core enforces this at registration time; incomplete drivers are rejected.

---

*End of Document*
