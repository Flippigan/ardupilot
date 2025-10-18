# ArduPilot SITL Instances - Comprehensive Research Documentation

**Research Date:** 2025-10-09
**Researcher:** Claude Code
**Component:** ArduPilot SITL Instances (ardupilot1-5)
**Project:** Multi-Drone ROS2 Formation Control System

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Component Overview](#component-overview)
3. [System Architecture](#system-architecture)
4. [Data Flow Analysis](#data-flow-analysis)
5. [Port Allocation Matrix](#port-allocation-matrix)
6. [Build Configuration](#build-configuration)
7. [DDS Topics & Services](#dds-topics--services)
8. [Gazebo Integration](#gazebo-integration)
9. [Launch Sequence](#launch-sequence)
10. [File Locations](#file-locations)

---

## Executive Summary

The ArduPilot SITL (Software-In-The-Loop) instances form the **flight controller layer** of the multi-drone system. Each instance (`ardupilot1` through `ardupilot5`) is an independent, full-featured ArduCopter flight controller running in software simulation mode. These instances provide:

- **Autonomous flight control**: PID controllers, EKF state estimation, sensor fusion
- **Mission planning**: Waypoint navigation, guided mode, RTL (Return-To-Launch)
- **ROS2 integration**: Via DDS (Data Distribution Service) for telemetry and commands
- **Gazebo physics simulation**: Via JSON/UDP protocol for realistic flight dynamics
- **MAVLink communication**: For ground control station (GCS) connectivity

Each instance operates completely independently with unique ports and namespaces, enabling true multi-drone simulation without interference.

---

## Component Overview

### What is ArduPilot SITL?

ArduPilot SITL is a simulator that allows the ArduPilot flight controller firmware to run on a Linux computer without hardware. It simulates:

- **Vehicle physics**: Via external simulator (Gazebo)
- **Sensors**: IMU, GPS, barometer, compass (simulated by Gazebo, consumed by ArduPilot)
- **Actuators**: Motors, servos (commands sent to Gazebo for physics simulation)
- **Flight controller logic**: PID loops, EKF, mission planner, all ArduPilot features

### Directory Structure

```
ardu_ws/src/
├── ardupilot1/          # SITL Instance 1 (DDS Port 2019, FDM Port 9002)
├── ardupilot2/          # SITL Instance 2 (DDS Port 2029, FDM Port 9012)
├── ardupilot3/          # SITL Instance 3 (DDS Port 2039, FDM Port 9022)
├── ardupilot4/          # SITL Instance 4 (DDS Port 2049, FDM Port 9032)
└── ardupilot5/          # SITL Instance 5 (DDS Port 2059, FDM Port 9042)
```

Each instance is a **complete copy** of the ArduPilot source code, compiled independently with unique configuration parameters.

### Key Components Per Instance

Each ArduPilot instance contains:

- **Source Code**: Complete ArduPilot firmware (same for all instances)
- **Build Artifacts**: `build/sitl/bin/arducopter` - The SITL executable
- **Libraries**: `libraries/AP_DDS/` - DDS client implementation
- **Submodules**:
  - `modules/Micro-XRCE-DDS-Client/` - DDS communication library
  - `modules/mavlink/` - MAVLink protocol definitions
- **ROS2 Tools**: `Tools/ros2/ardupilot_sitl/` - Launch files and config
- **Parameter Files**: Instance-specific configuration

---

## System Architecture

### Three-Layer Architecture

The multi-drone system consists of three interconnected layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROS2 Control Layer                           │
│  (Formation Control, Safety Monitor, Ground Station Nodes)     │
│                                                                 │
│  Topics: /ap_1/pose/filtered, /ap_2/pose/filtered, etc.       │
└───────────────────────┬─────────────────────────────────────────┘
                        │ DDS/UDP (Ports 2019, 2029, ...)
┌───────────────────────┴─────────────────────────────────────────┐
│              Flight Controller Layer (ArduPilot SITL)          │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │SITL 1    │  │SITL 2    │  │SITL 3    │  │SITL 4    │  ... │
│  │Instance 1│  │Instance 2│  │Instance 3│  │Instance 4│      │
│  │Port 9002 │  │Port 9012 │  │Port 9022 │  │Port 9032 │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
└───────┼────────────┼────────────┼────────────┼─────────────────┘
        │ JSON/UDP   │            │            │
┌───────┴────────────┴────────────┴────────────┴─────────────────┐
│                   Physics Simulation Layer (Gazebo)            │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │iris_9002 │  │iris_9012 │  │iris_9022 │  │iris_9032 │  ... │
│  │ArduPilot │  │ArduPilot │  │ArduPilot │  │ArduPilot │      │
│  │Plugin    │  │Plugin    │  │Plugin    │  │Plugin    │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                                 │
│  Simulates: Physics, Collisions, Sensors, Motors              │
└─────────────────────────────────────────────────────────────────┘
```

### Role of ArduPilot SITL

ArduPilot SITL serves as the **autonomous flight controller** for each drone:

1. **Receives sensor data** from Gazebo (IMU, GPS, barometer, compass)
2. **Runs flight algorithms**: EKF for state estimation, PID for control
3. **Generates motor commands** based on flight mode and pilot input
4. **Sends motor commands** back to Gazebo for physics simulation
5. **Publishes telemetry** to ROS2 via DDS (position, velocity, battery, status)
6. **Receives commands** from ROS2 (velocity commands, GPS waypoints)
7. **Provides services** for arming, mode switching, takeoff

---

## Data Flow Analysis

### Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ROS2 Network (DDS)                          │
│                                                                     │
│  Formation Control ──► /ap_1/cmd_vel ──┐                          │
│  Ground Station    ──► /ap_1/cmd_gps_pose                         │
│  Safety Monitor    ◄── /ap_1/pose/filtered ◄──┐                   │
│                    ◄── /ap_1/battery            │                   │
└─────────────────────────────────────────────────┼───────────────────┘
                                                  │
                     ┌────────────────────────────┴──────────────┐
                     │      Micro-ROS Agent (DDS Bridge)         │
                     │      Namespace: ap_1                       │
                     │      Transport: UDP                        │
                     │      Port: 2019                            │
                     └────────────────────────────────────────────┘
                                        ▲
                                        │ UDP (localhost:2019)
                                        │ eProsima Micro XRCE-DDS
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 ArduPilot SITL Instance 1                           │
│                 /home/finn/.../ardupilot1/                          │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │  ArduCopter Flight Controller                             │    │
│  │                                                            │    │
│  │  • Extended Kalman Filter (EKF)                           │    │
│  │  • PID Controllers (Roll, Pitch, Yaw, Altitude)           │    │
│  │  • Mission Planner / Guided Mode                          │    │
│  │  • Sensor Fusion                                           │    │
│  └───────────────────────────────────────────────────────────┘    │
│                          ▲              │                           │
│                          │ Sensors      │ Motor Commands            │
│  ┌───────────────────────┴──────────────▼─────────────────────┐   │
│  │       AP_DDS Client (Micro XRCE-DDS Client)                │   │
│  │       • Publishes: pose, velocity, battery, status, etc.   │   │
│  │       • Subscribes: cmd_vel, cmd_gps_pose, joy             │   │
│  │       • Services: arm_motors, mode_switch, takeoff         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              JSON/UDP Physics Interface                     │   │
│  │              Port IN:  9002 (from Gazebo)                   │   │
│  │              Port OUT: 9003 (to Gazebo)                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                          ▲              │                           │
└──────────────────────────┼──────────────┼───────────────────────────┘
                           │ Sensor Data  │ PWM Commands
                           │ (JSON)       │ (JSON)
┌──────────────────────────┴──────────────▼───────────────────────────┐
│                       Gazebo Simulation                              │
│                       Model: iris_with_gimbal_9002                   │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │            ArduPilotPlugin                                  │    │
│  │            FDM Port IN:  9002                               │    │
│  │            FDM Port OUT: 9003                               │    │
│  │            Lock-step: Enabled                               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                          ▲              │                            │
│          ┌───────────────┘              └────────────┐              │
│          │ Read Sensors                      Apply Forces           │
│          │                                                           │
│  ┌───────▼────────────────────────────────────────▼──────────┐     │
│  │  Physics Engine (Gazebo)                                   │     │
│  │  • IMU Sensor                                              │     │
│  │  • GPS Sensor                                              │     │
│  │  • Barometer                                               │     │
│  │  • Magnetometer                                            │     │
│  │  • Motor Physics (4 rotors with lift/drag)                │     │
│  │  • Collision Detection                                     │     │
│  │  • Gravity, Aerodynamics                                   │     │
│  └────────────────────────────────────────────────────────────┘     │
│                          │                                           │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │       ros_gz_bridge (Gazebo → ROS2 Bridge)                 │    │
│  │       Namespace: iris_9002                                  │    │
│  │       Topics: /iris_9002/imu, /iris_9002/odometry, etc.    │    │
│  └────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
                    ROS2 Visualization
                    (RViz, Ground Truth)
```

### Data Entering ArduPilot SITL

**Source: Gazebo Physics Simulation**
**Method: JSON over UDP**
**Port: 9002 (IN) per instance**

Data sent from Gazebo to ArduPilot every physics step (1000 Hz):
- **IMU Data**:
  - Linear acceleration (m/s²): x, y, z
  - Angular velocity (rad/s): roll rate, pitch rate, yaw rate
- **GPS Data**:
  - Latitude, longitude, altitude
  - Ground speed, heading
- **Barometer**:
  - Absolute pressure
  - Temperature
- **Magnetometer**:
  - Magnetic field vector (µT): x, y, z
- **Airspeed** (if enabled)
- **Rangefinder** (if enabled)

**Data Format**: JSON string, lock-step synchronized (Gazebo waits for ArduPilot response)

### Data Leaving ArduPilot SITL

#### To Gazebo (Physics Commands)
**Method: JSON over UDP**
**Port: 9003 (OUT) per instance**

Motor commands sent every control loop (400 Hz typical):
- **Motor PWM values** (channels 0-3):
  - Range: 1100-1900 µs
  - Mapped to motor velocities by ArduPilotPlugin
- **Gimbal commands** (channels 8-10):
  - Roll, pitch, yaw servo positions
- **Timestamp**: For lock-step synchronization

#### To ROS2 (Telemetry)
**Method: DDS over UDP**
**Port: 2019, 2029, 2039, 2049, 2059 (per instance)**
**Namespace: /ap_1/, /ap_2/, /ap_3/, /ap_4/, /ap_5/**

Published topics (ArduPilot → ROS2):
- `/ap_N/time` - System timestamp (builtin_interfaces/msg/Time)
- `/ap_N/clock` - Simulation clock (rosgraph_msgs/msg/Clock)
- `/ap_N/pose/filtered` - EKF position estimate (geometry_msgs/msg/PoseStamped)
- `/ap_N/twist/filtered` - EKF velocity estimate (geometry_msgs/msg/TwistStamped)
- `/ap_N/navsat` - GPS position (sensor_msgs/msg/NavSatFix)
- `/ap_N/imu/experimental/data` - IMU data (sensor_msgs/msg/Imu)
- `/ap_N/battery` - Battery state (sensor_msgs/msg/BatteryState)
- `/ap_N/status` - Vehicle status (ardupilot_msgs/msg/Status)
- `/ap_N/geopose/filtered` - Global pose (geographic_msgs/msg/GeoPoseStamped)
- `/ap_N/tf_static` - Static transforms (tf2_msgs/msg/TFMessage)

Subscribed topics (ROS2 → ArduPilot):
- `/ap_N/cmd_vel` - Velocity commands (geometry_msgs/msg/TwistStamped)
- `/ap_N/cmd_gps_pose` - GPS waypoint commands (ardupilot_msgs/msg/GlobalPosition)
- `/ap_N/joy` - RC override commands (sensor_msgs/msg/Joy)
- `/ap_N/tf` - Dynamic transforms (tf2_msgs/msg/TFMessage)

Services (ROS2 ⟷ ArduPilot):
- `/ap_N/arm_motors` - Arm/disarm motors
- `/ap_N/mode_switch` - Change flight mode
- `/ap_N/prearm_check` - Check if vehicle is armable
- `/ap_N/experimental/takeoff` - Initiate takeoff
- `/ap_N/set_parameters` - Set ArduPilot parameters
- `/ap_N/get_parameters` - Get ArduPilot parameters

#### To MAVProxy (Ground Control Station)
**Method: MAVLink over TCP**
**Port: 5760, 5770, 5780, 5790, 5800 (per instance)**

MAVLink telemetry stream for:
- Mission Planner
- QGroundControl
- MAVProxy CLI
- Custom GCS applications

Additional UDP outputs on ports 14550, 14560, 14570, 14580, 14590.

---

## Port Allocation Matrix

### Complete Port Mapping

| Instance | FDM IN | FDM OUT | DDS Port | TCP MAVLink | UDP MAVLink | SITL Port | ROS2 Namespace |
|----------|--------|---------|----------|-------------|-------------|-----------|----------------|
| 1        | 9002   | 9003    | 2019     | 5760        | 14550       | 5501      | /ap_1/         |
| 2        | 9012   | 9013    | 2029     | 5770        | 14560       | 5511      | /ap_2/         |
| 3        | 9022   | 9023    | 2039     | 5780        | 14570       | 5521      | /ap_3/         |
| 4        | 9032   | 9033    | 2049     | 5790        | 14580       | 5531      | /ap_4/         |
| 5        | 9042   | 9043    | 2059     | 5800        | 14590       | 5541      | /ap_5/         |

### Port Purposes

- **FDM (Flight Dynamics Model) Ports**:
  - **IN**: Receives sensor data from Gazebo ArduPilotPlugin (JSON format)
  - **OUT**: Sends motor commands to Gazebo ArduPilotPlugin (JSON format)
  - Base port pattern: 9002 + (instance-1)*10

- **DDS Port**:
  - UDP port for Micro-ROS Agent communication
  - Handles all ROS2 topic/service traffic
  - Base port: 2019 + (instance-1)*10

- **TCP MAVLink**:
  - Primary MAVProxy connection
  - Used for GCS console, mission planning
  - Base port: 5760 + (instance-1)*10

- **UDP MAVLink**:
  - Secondary GCS output
  - Typically for QGroundControl
  - Base port: 14550 + (instance-1)*10

- **SITL Port**:
  - Internal ArduPilot SITL status port
  - Used by sim_vehicle.py for control
  - Base port: 5501 + (instance-1)*10

### Port Calculation Formula

ArduPilot uses the `--instance N` parameter to automatically offset all ports:

```
Base Port + (Instance * 10)
```

For example, Instance 3:
- FDM IN: 9002 + (3-1)*10 = 9022
- DDS: 2019 + (3-1)*10 = 2039
- MAVLink TCP: 5760 + (3-1)*10 = 5780

---

## Build Configuration

### How to Build ArduPilot SITL with DDS

Each instance was built using the `waf` build system:

```bash
cd /home/finn/Documents/ardu_ws/src/ardupilot1

# Configure for SITL with DDS enabled
./waf configure --board sitl --enable-dds

# Build ArduCopter
./waf copter
```

### Build Output

The build produces:
- **Executable**: `build/sitl/bin/arducopter`
- **Libraries**: Compiled AP_DDS, Micro-XRCE-DDS-Client
- **Size**: ~100MB per build directory

### DDS Compilation Details

When `--enable-dds` is used:
1. Submodule `modules/Micro-XRCE-DDS-Client/` is compiled
2. Library `libraries/AP_DDS/` is compiled with DDS support
3. IDL files in `libraries/AP_DDS/Idl/` are processed
4. DDS client is linked into the ArduCopter binary

### Build Dependencies

Required for DDS compilation:
- **Micro-XRCE-DDS-Gen**: IDL code generator
- **Python 3**: For build scripts
- **GCC/G++**: Compiler toolchain

### Instance-Specific Configuration

Each instance has unique parameter files:

**Base Configuration** (all instances):
```
ardupilot_gazebo/config/gazebo-iris-gimbal.parm
```
- Frame type: X-frame quadcopter
- Motor PWM range: 1100-1900
- Gimbal configuration: 3-DOF (roll, pitch, yaw)

**DDS Configuration** (per instance):
```
ardupilotN/Tools/ros2/ardupilot_sitl/config/default_params/dds_udp_instanceN.parm
```
Example for Instance 1:
```
DDS_ENABLE 1
DDS_UDP_PORT 2019
```

---

## DDS Topics & Services

### Topic Table Configuration

All DDS topics are defined in `libraries/AP_DDS/AP_DDS_Topic_Table.h`.

### Published Topics (ArduPilot → ROS2)

| ROS2 Topic | Message Type | QoS | Rate | Description |
|------------|--------------|-----|------|-------------|
| `/ap_N/time` | builtin_interfaces/msg/Time | Reliable, Volatile | 50 Hz | System time |
| `/ap_N/clock` | rosgraph_msgs/msg/Clock | Reliable, Volatile | 50 Hz | Simulation clock |
| `/ap_N/pose/filtered` | geometry_msgs/msg/PoseStamped | Best Effort, Volatile | 50 Hz | EKF position/orientation |
| `/ap_N/twist/filtered` | geometry_msgs/msg/TwistStamped | Best Effort, Volatile | 50 Hz | EKF linear/angular velocity |
| `/ap_N/navsat` | sensor_msgs/msg/NavSatFix | Best Effort, Volatile | 10 Hz | GPS position (lat/lon/alt) |
| `/ap_N/imu/experimental/data` | sensor_msgs/msg/Imu | Best Effort, Volatile | 50 Hz | IMU acceleration/gyro |
| `/ap_N/battery` | sensor_msgs/msg/BatteryState | Best Effort, Volatile | 1 Hz | Battery voltage/current/% |
| `/ap_N/status` | ardupilot_msgs/msg/Status | Reliable, Transient Local | 1 Hz | Armed, mode, GPS fix |
| `/ap_N/geopose/filtered` | geographic_msgs/msg/GeoPoseStamped | Best Effort, Volatile | 10 Hz | Global position with orientation |
| `/ap_N/tf_static` | tf2_msgs/msg/TFMessage | Reliable, Transient Local | Latched | Static sensor transforms |
| `/ap_N/gps_global_origin/filtered` | geographic_msgs/msg/GeoPointStamped | Best Effort, Volatile | 1 Hz | GPS origin point |
| `/ap_N/goal_lla` | geographic_msgs/msg/GeoPointStamped | Reliable, Transient Local | On change | Current navigation goal |
| `/ap_N/airspeed` | geometry_msgs/msg/Vector3Stamped | Best Effort, Volatile | 10 Hz | Airspeed vector |

### Subscribed Topics (ROS2 → ArduPilot)

| ROS2 Topic | Message Type | QoS | Description |
|------------|--------------|-----|-------------|
| `/ap_N/cmd_vel` | geometry_msgs/msg/TwistStamped | Best Effort, Volatile | Velocity commands in body frame |
| `/ap_N/cmd_gps_pose` | ardupilot_msgs/msg/GlobalPosition | Best Effort, Volatile | GPS waypoint target (GUIDED mode) |
| `/ap_N/joy` | sensor_msgs/msg/Joy | Best Effort, Volatile | RC override (8 channels, -1 to 1) |
| `/ap_N/tf` | tf2_msgs/msg/TFMessage | Best Effort, Volatile | Dynamic transforms |

### Services (ROS2 ⟷ ArduPilot)

All services use `Reliable, Transient Local` QoS.

| ROS2 Service | Service Type | Description |
|--------------|--------------|-------------|
| `/ap_N/arm_motors` | ardupilot_msgs/srv/ArmMotors | Arm or disarm the vehicle |
| `/ap_N/mode_switch` | ardupilot_msgs/srv/ModeSwitch | Switch flight mode (GUIDED, LOITER, RTL, etc.) |
| `/ap_N/prearm_check` | std_srvs/srv/Trigger | Check if vehicle is ready to arm |
| `/ap_N/experimental/takeoff` | ardupilot_msgs/srv/Takeoff | Command takeoff to specified altitude |
| `/ap_N/set_parameters` | rcl_interfaces/srv/SetParameters | Set ArduPilot parameters at runtime |
| `/ap_N/get_parameters` | rcl_interfaces/srv/GetParameters | Query ArduPilot parameter values |

### DDS Communication Architecture

```
ArduPilot SITL          Micro-ROS Agent           ROS2 Nodes
    (DDS Client)         (DDS Bridge)             (Subscribers)
       │                      │                        │
       │  UDP Port 2019      │                        │
       │─────────────────────▶│                        │
       │  XRCE Protocol      │                        │
       │  (Micro-CDR)        │                        │
       │                      │  DDS/RTPS              │
       │                      │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶│
       │                      │  (Discovery)           │
       │                      │                        │
       │                      │  Topic Data            │
       │                      │──────────────────────▶│
       │                      │  /ap_1/pose/filtered   │
```

The Micro-ROS Agent acts as a bridge:
1. Receives compact XRCE messages from ArduPilot (UDP)
2. Translates to full DDS/RTPS format
3. Publishes on ROS2 network using Fast-DDS

---

## Gazebo Integration

### ArduPilot Plugin Configuration

Each Gazebo model includes an `ArduPilotPlugin` that bridges ArduPilot SITL with Gazebo physics.

**Configuration Example** (from `iris_with_gimbal_9002/model.sdf`):

```xml
<plugin name="ArduPilotPlugin" filename="ArduPilotPlugin">
  <!-- UDP Connection -->
  <fdm_addr>127.0.0.1</fdm_addr>
  <fdm_port_in>9002</fdm_port_in>
  <fdm_port_out>9003</fdm_port_out>

  <!-- Lock-step simulation -->
  <lock_step>1</lock_step>
  <connectionTimeoutMaxCount>5</connectionTimeoutMaxCount>

  <!-- Frame conversions -->
  <modelXYZToAirplaneXForwardZDown degrees="true">0 0 0 180 0 0</modelXYZToAirplaneXForwardZDown>
  <gazeboXYZToNED degrees="true">0 0 0 180 0 90</gazeboXYZToNED>

  <!-- Sensors -->
  <imuName>imu_link::imu_sensor</imuName>

  <!-- Motor control (channels 0-3) -->
  <control channel="0">
    <jointName>rotor_0_joint</jointName>
    <type>VELOCITY</type>
    <multiplier>838</multiplier>  <!-- PWM to RPM -->
    <servo_min>1100</servo_min>
    <servo_max>1900</servo_max>
  </control>
  <!-- ... channels 1, 2, 3 for other motors ... -->

  <!-- Gimbal control (channels 8-10) -->
  <control channel="8">
    <jointName>roll_joint</jointName>
    <type>COMMAND</type>
  </control>
  <!-- ... channels 9, 10 for pitch/yaw ... -->
</plugin>
```

### Lock-Step Synchronization

With `<lock_step>1</lock_step>`:
1. Gazebo pauses physics after each step
2. Sends sensor data to ArduPilot via UDP
3. Waits for motor command response
4. Applies motor commands to physics
5. Advances simulation time

This ensures **deterministic simulation** where simulation time = real time * speedup.

### Sensor Data Flow (Gazebo → ArduPilot)

Sensors simulated by Gazebo:
- **IMU** (`imu_link::imu_sensor`):
  - Linear acceleration (body frame)
  - Angular velocity (body frame)
  - Includes realistic noise models

- **GPS** (`base_link::navsat_sensor`):
  - Latitude, longitude, altitude (WGS84)
  - Ground speed, heading
  - Configurable accuracy/noise

- **Barometer** (`base_link::air_pressure_sensor`):
  - Absolute pressure
  - Altitude estimate

- **Magnetometer** (`base_link::magnetometer_sensor`):
  - Magnetic field vector (NED frame)
  - Earth magnetic field model

- **Ground Truth** (for debugging):
  - Published to ROS2 via `ros_gz_bridge`
  - Not sent to ArduPilot (cheating)

### Motor Command Flow (ArduPilot → Gazebo)

ArduPilot sends PWM values (1100-1900 µs) for 4 motors:
1. **ArduPilot** calculates desired motor speeds based on:
   - Pilot input (or autonomous commands)
   - PID controller output
   - Motor mixing matrix

2. **JSON Message** sent via UDP:
   ```json
   {
     "timestamp": 1234567890.123,
     "motors": [1500, 1500, 1500, 1500]
   }
   ```

3. **ArduPilotPlugin** receives and:
   - Maps PWM (1100-1900) to velocity (-838 to +838 RPM)
   - Applies force to rotor joints
   - Accounts for rotor direction (CW/CCW)

4. **Gazebo Physics** calculates:
   - Thrust from each rotor (lift/drag equations)
   - Torques on vehicle body
   - Resulting motion (position, velocity, acceleration)

---

## Launch Sequence

### Multi-Drone Launch Timeline

The launch file `iris_multi_uav.launch.py` starts components in a specific order with delays:

```
T=0s    : Launch SITL Instance 1 + Micro-ROS Agent 1 + MAVProxy 1
T=7s    : Launch SITL Instance 2 + Micro-ROS Agent 2 + MAVProxy 2
T=14s   : Launch SITL Instance 3 + Micro-ROS Agent 3 + MAVProxy 3
T=21s   : Launch SITL Instance 4 + Micro-ROS Agent 4 + MAVProxy 4
T=28s   : Launch SITL Instance 5 + Micro-ROS Agent 5 + MAVProxy 5
T=35s   : Launch Gazebo (server + gui)
T=35s   : Launch robot_state_publisher nodes (5 instances)
T=35s   : Launch ros_gz_bridge nodes (5 instances)
T=35s+  : Launch TF relays (conditional, after bridges start)
```

**Delay Rationale**: 7-second delays between SITL instances ensure:
- Full DDS participant initialization
- Unique discovery ports don't conflict
- Stable Micro-ROS Agent connections

### Per-Instance Launch Process

For each ArduPilot instance, the launch sequence is:

**1. Micro-ROS Agent Launch**
```bash
micro_ros_agent udp4 --port 2019 --namespace ap_1 --middleware dds --verbose 4
```
- Starts listening on UDP port
- Waits for ArduPilot SITL to connect

**2. ArduPilot SITL Launch**
```bash
/home/finn/Documents/ardu_ws/src/ardupilot1/build/sitl/bin/arducopter \
  --model json \
  --speedup 1 \
  --slave 0 \
  --sim-address=127.0.0.1 \
  --instance 1 \
  --defaults gazebo-iris-gimbal.parm,dds_udp_instance1.parm \
  --synthetic-clock
```

Key parameters:
- `--model json`: Use JSON protocol for external simulator (Gazebo)
- `--speedup 1`: Real-time simulation (no speedup)
- `--instance 1`: Instance number (offsets all ports by 10*instance)
- `--defaults ...`: Load parameter files (DDS config, frame type, etc.)
- `--synthetic-clock`: Use simulation time, not system time

**3. MAVProxy Launch**
```bash
mavproxy.py \
  --out 127.0.0.1:14550 \
  --out 127.0.0.1:14551 \
  --master tcp:127.0.0.1:5760 \
  --sitl 127.0.0.1:5501 \
  --non-interactive
```

- Connects to ArduPilot SITL via TCP (port 5760)
- Provides MAVLink interface for GCS tools
- Runs in non-interactive mode (no console prompts)

### Startup Sequence Within ArduPilot

When ArduPilot SITL starts:

1. **Initialization Phase** (~3 seconds):
   - Load parameter files
   - Initialize subsystems (EKF, controllers, etc.)
   - Set up sensor backends
   - Configure DDS client

2. **DDS Connection Phase** (~1 second):
   - AP_DDS_Client attempts connection to Micro-ROS Agent
   - Retries with exponential backoff
   - Creates DDS DataWriters (publishers) and DataReaders (subscribers)
   - Publishes discovery information

3. **Simulator Connection Phase** (~1 second):
   - ArduPilot sends JSON "ready" message to Gazebo
   - Waits for first sensor data packet
   - Validates FDM connection
   - If no response after 5 seconds, logs warning

4. **Ready State** (T+5s):
   - ArduPilot prints: "AP: ArduPilot Ready"
   - Begins publishing ROS2 topics
   - Accepts commands from GCS and ROS2
   - Ready to arm (if pre-arm checks pass)

### Gazebo Startup

After all SITL instances are running, Gazebo starts:

1. **Gazebo Server** (T=35s):
   ```bash
   gz sim -v4 -s -r iris_multiuav.sdf
   ```
   - `-s`: Server only (no GUI yet)
   - `-r`: Start running immediately
   - Loads world with 5 iris models

2. **Gazebo GUI** (T=35s, parallel):
   ```bash
   gz sim -v4 -g
   ```
   - `-g`: GUI only
   - Connects to existing server

3. **ArduPilot Plugin Initialization** (per model):
   - Plugin waits for UDP connection from ArduPilot SITL
   - Validates FDM port configuration
   - Begins lock-step simulation loop

### Verification Commands

After launch completes, verify all systems:

```bash
# Check ROS2 topics
ros2 topic list | grep /ap_

# Check all 5 instances are publishing
ros2 topic hz /ap_1/pose/filtered &
ros2 topic hz /ap_2/pose/filtered &
ros2 topic hz /ap_3/pose/filtered &
ros2 topic hz /ap_4/pose/filtered &
ros2 topic hz /ap_5/pose/filtered &

# Check services
ros2 service list | grep /ap_

# Check Gazebo models
gz model --list

# Check MAVLink connections
mavproxy.py --master tcp:127.0.0.1:5760 --console
```

---

## File Locations

### Source Code

| Path | Description |
|------|-------------|
| `/home/finn/Documents/ardu_ws/src/ardupilot1/` | Instance 1 source tree |
| `/home/finn/Documents/ardu_ws/src/ardupilot2/` | Instance 2 source tree |
| `/home/finn/Documents/ardu_ws/src/ardupilot3/` | Instance 3 source tree |
| `/home/finn/Documents/ardu_ws/src/ardupilot4/` | Instance 4 source tree |
| `/home/finn/Documents/ardu_ws/src/ardupilot5/` | Instance 5 source tree |

### Build Artifacts

| Path | Description |
|------|-------------|
| `ardupilotN/build/sitl/bin/arducopter` | SITL executable |
| `ardupilotN/build/sitl/libraries/` | Compiled libraries |
| `ardupilotN/build/sitl/modules/` | Compiled submodules |

### Configuration Files

| Path | Description |
|------|-------------|
| `ardupilotN/Tools/ros2/ardupilot_sitl/config/default_params/dds_udp_instanceN.parm` | DDS config for instance N |
| `ardupilot_gazebo/config/gazebo-iris-gimbal.parm` | Base frame configuration |
| `ardupilot_gz_bringup/config/iris_bridge.yaml` | Gazebo-ROS bridge config |

### Launch Files

| Path | Description |
|------|-------------|
| `ardupilot_gz_bringup/launch/iris_multi_uav.launch.py` | Main multi-drone launch |
| `ardupilotN/Tools/ros2/ardupilot_sitl/launch/sitl_dds_udp.launch.py` | Single SITL+DDS launch |
| `ardupilotN/Tools/ros2/ardupilot_sitl/launch/sitl.launch.py` | SITL-only launch |
| `ardupilotN/Tools/ros2/ardupilot_sitl/launch/micro_ros_agent.launch.py` | Micro-ROS Agent launch |
| `ardupilotN/Tools/ros2/ardupilot_sitl/launch/mavproxy.launch.py` | MAVProxy launch |

### DDS Implementation

| Path | Description |
|------|-------------|
| `ardupilotN/libraries/AP_DDS/` | DDS client library |
| `ardupilotN/libraries/AP_DDS/AP_DDS_Client.cpp` | Main DDS client implementation |
| `ardupilotN/libraries/AP_DDS/AP_DDS_Topic_Table.h` | Topic definitions |
| `ardupilotN/libraries/AP_DDS/AP_DDS_Service_Table.h` | Service definitions |
| `ardupilotN/libraries/AP_DDS/Idl/` | IDL message definitions |
| `ardupilotN/modules/Micro-XRCE-DDS-Client/` | eProsima DDS client (submodule) |

### Gazebo Models

| Path | Description |
|------|-------------|
| `ardupilot_gazebo/models/iris_with_gimbal_9002/` | Model for drone 1 |
| `ardupilot_gazebo/models/iris_with_gimbal_9012/` | Model for drone 2 |
| `ardupilot_gazebo/models/iris_with_gimbal_9022/` | Model for drone 3 |
| `ardupilot_gazebo/models/iris_with_gimbal_9032/` | Model for drone 4 |
| `ardupilot_gazebo/models/iris_with_gimbal_9042/` | Model for drone 5 |
| `ardupilot_gazebo/worlds/iris_multiuav.sdf` | Gazebo world definition |

---

## Summary

The ArduPilot SITL instances form the **intelligent flight control layer** of the multi-drone system. Each instance:

1. **Receives** sensor data from Gazebo via JSON/UDP (ports 9002, 9012, etc.)
2. **Processes** data through EKF, PID controllers, mission planner
3. **Generates** motor commands based on flight mode and control inputs
4. **Sends** motor commands back to Gazebo via JSON/UDP (ports 9003, 9013, etc.)
5. **Publishes** telemetry to ROS2 via DDS/UDP (ports 2019, 2029, etc.)
6. **Subscribes** to ROS2 commands (velocity, waypoints, RC override)
7. **Provides** ROS2 services (arm, mode switch, takeoff)
8. **Communicates** with GCS via MAVLink/TCP (ports 5760, 5770, etc.)

The system achieves **complete isolation** between instances through:
- Unique UDP ports (no conflicts)
- Separate ROS2 namespaces (/ap_1/, /ap_2/, etc.)
- Independent process spaces
- Instance-based port offset calculation

This architecture enables **realistic multi-drone simulation** where each drone has:
- Autonomous flight control (PID, EKF, mission planning)
- Independent failures and behavior
- Separate communication channels
- Isolated state and parameters

---

**Document Version:** 1.0
**Last Updated:** 2025-10-09
**Author:** Claude Code (Anthropic)
