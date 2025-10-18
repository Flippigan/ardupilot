# ArduPilot SITL - Base Repository

**Repository Type:** Base/Template Repository (NOT part of active swarm)
**Purpose:** Source code reference and development base for SITL instances
**Active Instances:** ardupilot1, ardupilot2, ardupilot3, ardupilot4, ardupilot5
**Parent Workspace:** `/home/finn/Documents/ardu_ws`
**Last Updated:** 2025-10-09

---

## Repository Overview

This is the **base ArduPilot repository** used as a reference and development source for the multi-drone formation control system. It is **NOT used for active simulation** - instead, it serves as the template from which the five instance directories (ardupilot1-5) are derived.

### Key Distinction

```
ardupilot/          ← THIS REPO (base/template, for development/reference)
ardupilot1/         ← Active instance 1 (DDS Port 2019, FDM Port 9002)
ardupilot2/         ← Active instance 2 (DDS Port 2029, FDM Port 9012)
ardupilot3/         ← Active instance 3 (DDS Port 2039, FDM Port 9022)
ardupilot4/         ← Active instance 4 (DDS Port 2049, FDM Port 9032)
ardupilot5/         ← Active instance 5 (DDS Port 2059, FDM Port 9042)
```

---

## Quick Reference

### Build SITL with DDS

```bash
# Configure for SITL with DDS enabled
./waf configure --board sitl --enable-dds

# Build ArduCopter
./waf copter

# Clean build (if needed)
./waf clean
```

### Test Build

```bash
# Verify the build succeeded
ls -lh build/sitl/bin/arducopter

# Run basic SITL test (standalone, no Gazebo)
build/sitl/bin/arducopter --model quad --console

# Test DDS compilation
./waf list | grep AP_DDS
```

### Check DDS Configuration

```bash
# View default DDS parameters
cat Tools/ros2/ardupilot_sitl/config/default_params/dds_udp.parm

# List DDS-related source files
ls -la libraries/AP_DDS/

# Check Micro-XRCE-DDS-Client submodule
git submodule status | grep Micro-XRCE-DDS
```

---

## System Architecture

### Three-Layer Multi-Drone System

```
┌─────────────────────────────────────────────────────┐
│         ROS2 Control Layer                          │
│  (Formation Control, Safety Monitor, Visualizer)   │
└──────────────────┬──────────────────────────────────┘
                   │ DDS/UDP (Micro-ROS Agent)
┌──────────────────┴──────────────────────────────────┐
│       Flight Controller Layer (ArduPilot SITL)     │
│  ardupilot1  ardupilot2  ardupilot3  ardupilot4  5 │
│  DDS: 2019   2029        2039        2049      2059│
│  FDM: 9002   9012        9022        9032      9042│
└──────────────────┬──────────────────────────────────┘
                   │ JSON/UDP (Sensor Data ⇄ Motor Commands)
┌──────────────────┴──────────────────────────────────┐
│         Physics Simulation (Gazebo)                 │
│  ArduPilotPlugin × 5 (one per drone)               │
└─────────────────────────────────────────────────────┘
```

### ArduPilot SITL Role

Each ArduPilot SITL instance acts as an **autonomous flight controller**:

1. **Receives** sensor data from Gazebo (IMU, GPS, barometer, compass)
2. **Processes** via EKF (Extended Kalman Filter) for state estimation
3. **Controls** using PID controllers (roll, pitch, yaw, altitude)
4. **Generates** motor PWM commands (1100-1900 µs)
5. **Sends** motor commands to Gazebo for physics simulation
6. **Publishes** telemetry to ROS2 (position, velocity, battery, status)
7. **Subscribes** to ROS2 commands (velocity, waypoints, RC override)
8. **Provides** ROS2 services (arm_motors, mode_switch, takeoff)
9. **Communicates** with ground control via MAVLink

---

## Port Configuration Overview

Each instance uses unique ports calculated by: **Base Port + (Instance - 1) × 10**

| Service | Base Port | Instance 1 | Instance 2 | Instance 3 | Instance 4 | Instance 5 |
|---------|-----------|------------|------------|------------|------------|------------|
| DDS | 2019 | 2019 | 2029 | 2039 | 2049 | 2059 |
| FDM IN | 9002 | 9002 | 9012 | 9022 | 9032 | 9042 |
| FDM OUT | 9003 | 9003 | 9013 | 9023 | 9033 | 9043 |
| TCP MAVLink | 5760 | 5760 | 5770 | 5780 | 5790 | 5800 |
| UDP MAVLink | 14550 | 14550 | 14560 | 14570 | 14580 | 14590 |
| SITL Control | 5501 | 5501 | 5511 | 5521 | 5531 | 5541 |

**Port Purposes:**
- **DDS**: ROS2 communication via Micro-ROS Agent
- **FDM**: Flight Dynamics Model (Gazebo ↔ ArduPilot sensor/motor data)
- **MAVLink**: Ground control station connectivity
- **SITL Control**: Internal sim_vehicle.py control interface

---

## Important File Locations

### Build Artifacts
- **Executable**: `build/sitl/bin/arducopter` - The SITL binary
- **Build Config**: `build/sitl/build.config.json` - Build configuration
- **Libraries**: `build/sitl/libraries/` - Compiled AP libraries

### DDS Implementation
- **Client Library**: `libraries/AP_DDS/` - DDS client implementation
  - `AP_DDS_Client.cpp` - Main DDS client logic
  - `AP_DDS_Topic_Table.h` - Topic definitions (publishers)
  - `AP_DDS_Service_Table.h` - Service definitions
- **IDL Definitions**: `libraries/AP_DDS/Idl/` - Message interface definitions
- **Submodule**: `modules/Micro-XRCE-DDS-Client/` - eProsima DDS library

### Configuration Files
- **DDS Default**: `Tools/ros2/ardupilot_sitl/config/default_params/dds_udp.parm`
- **DDS Serial**: `Tools/ros2/ardupilot_sitl/config/default_params/dds_serial.parm`
- **Frame Config**: `../ardupilot_gazebo/config/gazebo-iris-gimbal.parm` (workspace level)

### ROS2 Launch Files
- **SITL + DDS**: `Tools/ros2/ardupilot_sitl/launch/sitl_dds_udp.launch.py`
- **SITL Only**: `Tools/ros2/ardupilot_sitl/launch/sitl.launch.py`
- **Micro-ROS Agent**: `Tools/ros2/ardupilot_sitl/launch/micro_ros_agent.launch.py`
- **MAVProxy**: `Tools/ros2/ardupilot_sitl/launch/mavproxy.launch.py`

### Core Flight Controller Code
- **ArduCopter**: `ArduCopter/` - Main copter code
- **Libraries**: `libraries/` - AP subsystems
  - `AP_NavEKF*/` - Extended Kalman Filters
  - `AC_AttitudeControl/` - Attitude control
  - `AC_PosControl/` - Position control
  - `AP_Motors/` - Motor mixing and output
  - `AP_GPS/` - GPS drivers
  - `AP_InertialSensor/` - IMU drivers
  - `AP_Baro/` - Barometer drivers

### Submodules
- `modules/Micro-XRCE-DDS-Client/` - DDS communication
- `modules/mavlink/` - MAVLink protocol
- `modules/gbenchmark/` - Google benchmark (testing)

---

## ROS2 Interface (DDS Topics & Services)

### Published Topics (ArduPilot → ROS2)

**Namespace Pattern:** `/ap_{N}/` where N = instance number (1-5)

| Topic | Type | Rate | Description |
|-------|------|------|-------------|
| `pose/filtered` | geometry_msgs/PoseStamped | 50 Hz | EKF position/orientation (NED frame) |
| `twist/filtered` | geometry_msgs/TwistStamped | 50 Hz | EKF velocity (linear + angular) |
| `navsat` | sensor_msgs/NavSatFix | 10 Hz | GPS position (lat/lon/alt) |
| `imu/experimental/data` | sensor_msgs/Imu | 50 Hz | IMU acceleration + gyro |
| `battery` | sensor_msgs/BatteryState | 1 Hz | Battery voltage, current, percentage |
| `status` | ardupilot_msgs/Status | 1 Hz | Armed state, flight mode, GPS fix |
| `geopose/filtered` | geographic_msgs/GeoPoseStamped | 10 Hz | Global position with orientation |
| `tf_static` | tf2_msgs/TFMessage | Latched | Static sensor transforms |
| `clock` | rosgraph_msgs/Clock | 50 Hz | Simulation clock |

### Subscribed Topics (ROS2 → ArduPilot)

| Topic | Type | Description |
|-------|------|-------------|
| `cmd_vel` | geometry_msgs/TwistStamped | Velocity commands (body frame, GUIDED mode) |
| `cmd_gps_pose` | ardupilot_msgs/GlobalPosition | GPS waypoint target (GUIDED mode) |
| `joy` | sensor_msgs/Joy | RC override (8 channels, -1 to 1) |
| `tf` | tf2_msgs/TFMessage | Dynamic transforms |

### Services (ROS2 ⟷ ArduPilot)

| Service | Type | Description |
|---------|------|-------------|
| `arm_motors` | ardupilot_msgs/srv/ArmMotors | Arm/disarm vehicle motors |
| `mode_switch` | ardupilot_msgs/srv/ModeSwitch | Change flight mode (GUIDED, LOITER, RTL, etc.) |
| `prearm_check` | std_srvs/srv/Trigger | Check if vehicle is ready to arm |
| `experimental/takeoff` | ardupilot_msgs/srv/Takeoff | Command takeoff to altitude |
| `set_parameters` | rcl_interfaces/srv/SetParameters | Set ArduPilot parameters |
| `get_parameters` | rcl_interfaces/srv/GetParameters | Query parameter values |

**Full Topic List:** See `.claude/10092025 update claude docs/RESEARCH.md` §7

---

## Common Workflows

### Build After Source Code Changes

```bash
# 1. Clean if headers changed
./waf clean

# 2. Reconfigure (only if build system changed)
./waf configure --board sitl --enable-dds

# 3. Build
./waf copter

# 4. Verify
ls -lh build/sitl/bin/arducopter
```

### Add a New ROS2 Topic

1. **Define topic** in `libraries/AP_DDS/AP_DDS_Topic_Table.h`:
   ```cpp
   {
       .topic_id = DDS_TOPIC_MY_NEW_TOPIC,
       .pub_id = DDS_PUB_MY_NEW_TOPIC,
       .dw_id = DDS_DW_MY_NEW_TOPIC,
       .topic_profile_label = "my_new_topic",
       .dds_type_name = "geometry_msgs::msg::dds_::Vector3_"
   }
   ```

2. **Implement publisher** in `libraries/AP_DDS/AP_DDS_Client.cpp`:
   ```cpp
   bool AP_DDS_Client::publish_my_new_topic() {
       geometry_msgs_msg_Vector3 msg;
       // Populate msg fields
       return dds_write(&my_new_topic_writer, &msg);
   }
   ```

3. **Rebuild**: `./waf copter`

4. **Test**: `ros2 topic echo /ap_1/my_new_topic`

### Test DDS Without Gazebo

```bash
# Terminal 1: Start Micro-ROS Agent
ros2 run micro_ros_agent micro_ros_agent udp4 --port 2019 --namespace ap_test

# Terminal 2: Run SITL standalone
build/sitl/bin/arducopter \
  --model quad \
  --speedup 1 \
  --defaults Tools/ros2/ardupilot_sitl/config/default_params/dds_udp.parm \
  --synthetic-clock

# Terminal 3: Verify ROS2 topics
ros2 topic list | grep /ap_test/
ros2 topic echo /ap_test/pose/filtered --once
```

### Update Instance Repositories

```bash
# When changes are made to this base repository,
# propagate to instances as needed:

cd /home/finn/Documents/ardu_ws/src/ardupilot1
git pull origin master  # if instances are git repos

# Or manually copy specific files
cp ../ardupilot/libraries/AP_DDS/some_file.cpp libraries/AP_DDS/

# Rebuild the instance
./waf copter
```

---

## Troubleshooting

### Build Failures

**Problem:** `waf configure` fails with "DDS not found"
```
Checking for library Micro-XRCE-DDS-Client : not found
```

**Solution:**
```bash
# Initialize/update submodules
git submodule update --init --recursive

# Verify submodule exists
ls modules/Micro-XRCE-DDS-Client/

# Try configure again
./waf configure --board sitl --enable-dds
```

---

**Problem:** Compilation errors in `AP_DDS_Client.cpp`
```
error: 'dds_write' was not declared in this scope
```

**Solution:**
```bash
# Clean and rebuild
./waf clean
./waf configure --board sitl --enable-dds
./waf copter

# If still failing, check DDS submodule
cd modules/Micro-XRCE-DDS-Client
git status
cd ../..
```

---

**Problem:** Python version errors during build
```
Python version must be >= 3.6
```

**Solution:**
```bash
# Check Python version
python3 --version

# Ensure Python 3 is used
export PYTHON=python3
./waf configure --board sitl --enable-dds
```

---

### Runtime Issues (For Instance Testing)

**Problem:** No ROS2 topics appearing after starting SITL
```bash
ros2 topic list | grep /ap_1/
# (empty)
```

**Checklist:**
1. Is Micro-ROS Agent running? `ps aux | grep micro_ros_agent`
2. Is DDS enabled in parm file? `grep DDS_ENABLE <parm_file>`
3. Is DDS port correct? `grep DDS_UDP_PORT <parm_file>`
4. Check ArduPilot startup output for DDS connection messages
5. Check firewall: `sudo ufw status`

---

**Problem:** Port conflicts when running multiple instances
```
bind: Address already in use
```

**Solution:**
```bash
# Find process using port
lsof -i :2019

# Kill stuck processes
pkill arducopter
pkill micro_ros_agent

# Verify ports are free
netstat -an | grep 2019
```

---

### Submodule Issues

**Problem:** Submodules not initialized
```
fatal: Not a git repository: modules/Micro-XRCE-DDS-Client/.git
```

**Solution:**
```bash
# Initialize all submodules recursively
git submodule update --init --recursive

# Check status
git submodule status

# If submodule is detached or broken
git submodule sync
git submodule update --init --recursive
```

---

## Development Guidelines

### Code Style
- Follow [ArduPilot coding standards](https://ardupilot.org/dev/docs/coding-style.html)
- Use 4-space indentation (not tabs)
- Max line length: 120 characters
- Always run `./waf check` before committing

### Testing
- Test SITL standalone before integrating
- Verify ROS2 topics after DDS changes: `ros2 topic list`
- Check MAVLink compatibility with MAVProxy
- Run unit tests: `./waf check`

### Making Changes
1. **Branch Strategy**: Create feature branch from master
2. **Build Verification**: `./waf copter` must succeed
3. **Integration Test**: Test with instance directories if needed
4. **Documentation**: Update this CLAUDE.md if architecture changes

### Debugging Tools
- **GDB**: `gdb --args build/sitl/bin/arducopter <args>`
- **Logs**: `/tmp/ArduCopter.log` (default location)
- **Verbose**: Add `--log-level DEBUG` to SITL command
- **MAVProxy**: Real-time telemetry and command testing
- **ROS2 CLI**: `ros2 topic echo`, `ros2 service call`

---

## Integration with Multi-Drone System

### Relationship to Workspace

```
ardu_ws/
├── .claude/
│   └── CLAUDE.md                    ← Workspace-level documentation
├── src/
│   ├── ardupilot/                   ← THIS REPO (base/template)
│   │   └── .claude/
│   │       ├── CLAUDE.md            ← This file
│   │       ├── RESEARCH.md          ← Detailed technical analysis
│   │       └── PLAN.md              ← Documentation creation plan
│   ├── ardupilot1/                  ← Instance 1 (active)
│   ├── ardupilot2/                  ← Instance 2 (active)
│   ├── ardupilot3/                  ← Instance 3 (active)
│   ├── ardupilot4/                  ← Instance 4 (active)
│   ├── ardupilot5/                  ← Instance 5 (active)
│   ├── ardupilot_gazebo/            ← Gazebo models and plugins
│   ├── ardupilot_gz_bringup/        ← Multi-drone launch files
│   └── formation_control/           ← ROS2 formation control nodes
```

### Launch Integration

Multi-drone simulation uses: `ardupilot_gz/ardupilot_gz_bringup/launch/iris_multi_uav.launch.py`

This launch file:
1. Starts all 5 SITL instances (from ardupilot1-5 directories)
2. Starts 5 Micro-ROS Agent instances (one per namespace)
3. Launches Gazebo with 5 drone models
4. Starts ros_gz_bridge for Gazebo ↔ ROS2 communication
5. Optionally launches RViz for visualization

**Launch Command:**
```bash
ros2 launch ardupilot_gz_bringup iris_multi_uav.launch.py rviz:=true use_gz_tf:=true
```

Note: The package name is `ardupilot_gz_bringup` but the source path is `ardupilot_gz/ardupilot_gz_bringup/`.

### Communication Flow

Each ArduPilot instance communicates independently:

**Upstream (from Gazebo):**
- Gazebo ArduPilotPlugin → JSON/UDP → ArduPilot SITL
- Sensor data: IMU, GPS, barometer, compass

**Downstream (to ROS2):**
- ArduPilot SITL → DDS/UDP → Micro-ROS Agent → ROS2 Network
- Telemetry: pose, velocity, battery, status

**Control Input (from ROS2):**
- Formation Control → ROS2 Network → Micro-ROS Agent → ArduPilot SITL
- Commands: velocity, waypoints, arm/disarm, mode switch

**No Inter-Drone Communication:**
- Drones do NOT communicate directly with each other
- All coordination happens through ROS2 formation control nodes
- Centralized ground station architecture

---

## Flight Modes

ArduPilot supports multiple flight modes relevant to formation control:

| Mode | Description | Use Case |
|------|-------------|----------|
| **STABILIZE** | Manual attitude control, auto-level | Direct RC control |
| **ALT_HOLD** | Manual horizontal, auto altitude | Hovering in place |
| **LOITER** | GPS position hold | Maintain position |
| **GUIDED** | External navigation commands | **Formation control** |
| **AUTO** | Follow mission waypoints | Pre-planned missions |
| **RTL** | Return to launch point | Emergency return |
| **LAND** | Autonomous landing | End of mission |

**For Formation Control:** Use **GUIDED** mode
- Accepts velocity commands via `/ap_N/cmd_vel`
- Accepts GPS waypoints via `/ap_N/cmd_gps_pose`
- Provides full autonomous control from ROS2

**Mode Switch:**
```bash
# Via ROS2 service
ros2 service call /ap_1/mode_switch ardupilot_msgs/srv/ModeSwitch "{mode: 4}"
# mode: 4 = GUIDED

# Via MAVProxy
mavproxy.py --master tcp:127.0.0.1:5760
> mode GUIDED
```

---

## Key ArduPilot Parameters

Relevant parameters for DDS/ROS2 integration:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DDS_ENABLE` | 0 | Enable DDS (0=disabled, 1=enabled) |
| `DDS_UDP_PORT` | 2019 | UDP port for Micro-ROS Agent |
| `SYSID_THISMAV` | 1 | MAVLink system ID (unique per drone) |
| `FRAME_CLASS` | 1 | Frame type (1=quad, 2=hexa, etc.) |
| `FRAME_TYPE` | 1 | Frame configuration (1=X, 0=+) |
| `EKF_TYPE` | 3 | EKF version (3=EKF3, recommended) |
| `GPS_TYPE` | 1 | GPS driver (1=AUTO, 14=MAVLink) |
| `SERIAL1_PROTOCOL` | 2 | Serial port 1 protocol (2=MAVLink2) |

**View parameters in MAVProxy:**
```bash
mavproxy.py --master tcp:127.0.0.1:5760
> param show DDS_*
> param set DDS_ENABLE 1
> param set DDS_UDP_PORT 2019
```

**View parameters via ROS2:**
```bash
ros2 service call /ap_1/get_parameters rcl_interfaces/srv/GetParameters \
  "{names: ['DDS_ENABLE', 'DDS_UDP_PORT']}"
```

---

## References

### Project Documentation
- **Workspace CLAUDE.md**: `/home/finn/Documents/ardu_ws/.claude/CLAUDE.md` - System architecture overview
- **RESEARCH.md**: `.claude/10092025 update claude docs/RESEARCH.md` - Detailed technical analysis (838 lines)
- **PLAN.md**: `.claude/10092025 update claude docs/PLAN.md` - Documentation creation plan
- **Multi-Drone Launch**: `../ardupilot_gz/ardupilot_gz_bringup/launch/iris_multi_uav.launch.py` - Main launch file

### Upstream ArduPilot Documentation
- **SITL Overview**: https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html
- **DDS/ROS2**: https://ardupilot.org/dev/docs/ros2.html
- **Building ArduPilot**: https://ardupilot.org/dev/docs/building-the-code.html
- **Coding Style**: https://ardupilot.org/dev/docs/coding-style.html
- **MAVLink**: https://mavlink.io/en/

### External Dependencies
- **Gazebo ArduPilot Plugin**: https://github.com/ArduPilot/ardupilot_gazebo
- **Micro-XRCE-DDS**: https://micro-xrce-dds.docs.eprosima.com/
- **Micro-ROS Agent**: https://github.com/micro-ROS/micro-ROS-Agent
- **Fast-DDS**: https://fast-dds.docs.eprosima.com/

### Related ROS2 Packages
- **ardupilot_gazebo**: Gazebo simulation models and plugins
- **ardupilot_gz_bringup**: ROS2 launch files for multi-drone system
- **formation_control**: High-level formation control nodes
- **formation_msgs**: Custom message definitions for formation commands

---

## Quick Command Reference

### Build Commands
```bash
./waf configure --board sitl --enable-dds     # Configure with DDS
./waf copter                                   # Build ArduCopter
./waf clean                                    # Clean build
./waf distclean                                # Deep clean (removes build/)
./waf check                                    # Run unit tests
./waf list                                     # List build targets
```

### Git Commands
```bash
git submodule update --init --recursive        # Initialize submodules
git submodule status                           # Check submodule status
git submodule sync                             # Sync submodule URLs
```

### ROS2 Commands
```bash
ros2 topic list | grep /ap_                    # List ArduPilot topics
ros2 topic echo /ap_1/pose/filtered            # Monitor drone position
ros2 topic hz /ap_1/pose/filtered              # Check publish rate
ros2 service list | grep /ap_                  # List ArduPilot services
ros2 service call /ap_1/arm_motors ardupilot_msgs/srv/ArmMotors "{arm: true}"
```

### MAVProxy Commands
```bash
mavproxy.py --master tcp:127.0.0.1:5760        # Connect to SITL
> mode GUIDED                                   # Switch to guided mode
> arm throttle                                  # Arm motors
> takeoff 10                                    # Takeoff to 10m
> status                                        # Show vehicle status
> param show DDS_*                              # Show DDS parameters
```

### Process Management
```bash
ps aux | grep arducopter                       # Find SITL processes
pkill arducopter                               # Kill all SITL instances
lsof -i :2019                                  # Check port usage
netstat -an | grep 2019                        # Check port status
```

---

**Document Version:** 1.0
**Author:** Claude Code (Anthropic)
**Repository:** Base ArduPilot SITL (Template)
**Status:** Reference Documentation
