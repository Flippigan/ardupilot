# CLAUDE.md Creation Plan for ArduPilot SITL Instance

**Date:** 2025-10-09
**Purpose:** Design a concise, actionable CLAUDE.md for individual ArduPilot instances
**Scope:** Single instance repo (ardupilot1, ardupilot2, etc.)
**Based on:** RESEARCH.md comprehensive analysis + Claude Code memory best practices

---

## Executive Summary

This plan outlines the creation of a CLAUDE.md file for individual ArduPilot SITL instance repositories. The goal is to distill the comprehensive technical documentation in RESEARCH.md into a concise, immediately actionable reference that follows Claude Code best practices.

**Key Principle:** The CLAUDE.md should serve as a **quick reference guide** for working with a single ArduPilot instance, not duplicate the detailed research documentation.

---

## Analysis of Current Documentation

### RESEARCH.md Strengths
- ✅ Comprehensive technical coverage (838 lines)
- ✅ Excellent diagrams and data flow analysis
- ✅ Complete port allocation matrices
- ✅ Detailed DDS topic/service documentation
- ✅ Thorough launch sequence description

### RESEARCH.md Limitations for Daily Use
- ❌ Too detailed for quick reference
- ❌ Covers all 5 instances (not instance-specific)
- ❌ Heavy on architecture theory, light on practical commands
- ❌ Lacks troubleshooting guidance
- ❌ No common workflows or recipes

### Gap Analysis: What's Missing
1. **Instance-Specific Context**: Which instance is THIS repo? (1, 2, 3, 4, or 5?)
2. **Quick Command Reference**: Build, run, test commands
3. **Integration Points**: How does this instance connect to workspace?
4. **Troubleshooting**: Common issues and solutions
5. **Development Workflows**: Making changes, testing, debugging
6. **File Navigation**: Where to find specific functionality

---

## CLAUDE.md Design Principles

Following Claude Code best practices:

### 1. Be Specific and Precise
- ✓ Use exact port numbers for THIS instance
- ✓ Provide complete commands (not "run the build command...")
- ✓ Include actual file paths, not placeholders
- ✓ Specify concrete examples, not general patterns

### 2. Organizational Clarity
- ✓ Markdown headings for logical grouping
- ✓ Bullet points for scannable information
- ✓ Tables for structured data (ports, paths)
- ✓ Code blocks for commands
- ✓ Clear section hierarchy

### 3. Actionable Content
- ✓ Frequently used commands prominently placed
- ✓ Development workflows documented
- ✓ Troubleshooting steps included
- ✓ Common tasks have step-by-step instructions

### 4. Conciseness
- ✓ Target: 200-400 lines (vs 838 in RESEARCH.md)
- ✓ Link to RESEARCH.md for deep dives
- ✓ Avoid duplicating upstream docs
- ✓ Focus on "what Claude needs to know to work here"

### 5. Maintainability
- ✓ Version control friendly
- ✓ Easy to update when ports/config change
- ✓ Clear ownership and last-update date
- ✓ Modular sections that can be updated independently

---

## Recommended CLAUDE.md Structure

### Section 1: Repository Identity
**Purpose:** Immediately establish context

```markdown
# ArduPilot SITL Instance {N}

**Instance Number:** {1, 2, 3, 4, or 5}
**Role:** Flight controller layer for drone {N} in multi-drone formation system
**ROS2 Namespace:** /ap_{N}/
**Parent Workspace:** /home/finn/Documents/ardu_ws
```

**Content:**
- Instance number (critical!)
- Role in larger system
- Relationship to workspace-level CLAUDE.md
- Link to RESEARCH.md for architecture details

**Rationale:** Claude needs to know "which repo am I in?" immediately

---

### Section 2: Quick Reference
**Purpose:** Commands used 80% of the time

```markdown
## Quick Reference

### Build
```bash
./waf configure --board sitl --enable-dds
./waf copter
```

### Run SITL
```bash
build/sitl/bin/arducopter --model json --speedup 1 --instance {N} \
  --defaults {paths to parm files}
```

### Test Connection
```bash
ros2 topic list | grep /ap_{N}/
ros2 topic echo /ap_{N}/pose/filtered
```
```

**Content:**
- Build commands (configure + build)
- Run SITL standalone
- Verify ROS2 connection
- Check Gazebo connection
- MAVProxy connection

**Rationale:** Most common tasks should be copy-pastable

---

### Section 3: Port Configuration
**Purpose:** Instance-specific port mapping

```markdown
## Port Configuration (Instance {N})

| Service | Port | Purpose |
|---------|------|---------|
| FDM IN | {9002 + (N-1)*10} | Gazebo → ArduPilot sensor data |
| FDM OUT | {9003 + (N-1)*10} | ArduPilot → Gazebo motor commands |
| DDS | {2019 + (N-1)*10} | ROS2 communication via Micro-ROS Agent |
| TCP MAVLink | {5760 + (N-1)*10} | Ground control station |
| UDP MAVLink | {14550 + (N-1)*10} | QGroundControl |
| SITL Control | {5501 + (N-1)*10} | sim_vehicle.py control |
```

**Content:**
- Complete port table with calculated values
- Brief explanation of each port's purpose
- Port conflict troubleshooting note

**Rationale:** Ports are instance-specific and critical for debugging

---

### Section 4: Integration Points
**Purpose:** How this instance connects to the system

```markdown
## Integration with Multi-Drone System

### Upstream (Gazebo Physics)
- **Model:** iris_with_gimbal_{port}
- **Plugin:** ArduPilotPlugin
- **Data Flow:** Sensors IN (IMU, GPS, baro) → Motor Commands OUT

### Downstream (ROS2 Control Layer)
- **Published Topics:** /ap_{N}/pose/filtered, /battery, /status (see DDS Topics)
- **Subscribed Topics:** /ap_{N}/cmd_vel, /cmd_gps_pose
- **Services:** /ap_{N}/arm_motors, /mode_switch

### Parallel Components
- **Other Instances:** ardupilot{1-5} (isolated, no direct communication)
- **Workspace Launch:** iris_multi_uav.launch.py coordinates all instances
```

**Content:**
- Gazebo integration (specific model name)
- ROS2 topics/services (brief list, link to full table)
- Relationship to other instances
- Launch file coordination

**Rationale:** Understanding context prevents Claude from making changes that break integration

---

### Section 5: Key File Locations
**Purpose:** Navigate to important files quickly

```markdown
## Important Files

### Build Artifacts
- **Executable:** `build/sitl/bin/arducopter` - The SITL binary
- **Build Config:** `build/sitl/build.config.json`

### Configuration
- **DDS Config:** `Tools/ros2/ardupilot_sitl/config/default_params/dds_udp_instance{N}.parm`
- **Base Params:** (inherited from ardupilot_gazebo)

### Source Code
- **DDS Implementation:** `libraries/AP_DDS/`
  - `AP_DDS_Client.cpp` - Main client logic
  - `AP_DDS_Topic_Table.h` - Topic definitions
- **Flight Controller:** `libraries/AC_*` (ArduCopter subsystems)
- **EKF:** `libraries/AP_NavEKF*/`

### Launch Files
- **SITL + DDS:** `Tools/ros2/ardupilot_sitl/launch/sitl_dds_udp.launch.py`
- **Micro-ROS Agent:** `Tools/ros2/ardupilot_sitl/launch/micro_ros_agent.launch.py`

### Submodules
- `modules/Micro-XRCE-DDS-Client/` - DDS communication
- `modules/mavlink/` - MAVLink protocol
```

**Content:**
- Categorized file paths (build, config, source, launch)
- Brief description of each file's purpose
- Instance-specific paths vs shared paths

**Rationale:** Helps Claude locate files without searching

---

### Section 6: DDS Topics & Services
**Purpose:** ROS2 API reference

```markdown
## ROS2 Interface (Namespace: /ap_{N}/)

### Key Published Topics
| Topic | Type | Rate | Description |
|-------|------|------|-------------|
| `pose/filtered` | PoseStamped | 50 Hz | EKF position/orientation |
| `twist/filtered` | TwistStamped | 50 Hz | EKF velocity |
| `battery` | BatteryState | 1 Hz | Battery status |
| `status` | ardupilot_msgs/Status | 1 Hz | Armed state, flight mode |

### Key Subscribed Topics
| Topic | Type | Description |
|-------|------|-------------|
| `cmd_vel` | TwistStamped | Velocity commands (GUIDED mode) |
| `cmd_gps_pose` | GlobalPosition | GPS waypoint target |

### Services
| Service | Type | Description |
|---------|------|-------------|
| `arm_motors` | ArmMotors | Arm/disarm vehicle |
| `mode_switch` | ModeSwitch | Change flight mode |

**Full Topic List:** See RESEARCH.md §7 "DDS Topics & Services"
```

**Content:**
- Top 5-10 most used topics
- All services (there aren't many)
- Link to comprehensive list in RESEARCH.md
- Message types and rates

**Rationale:** Claude needs ROS2 API but not all 30+ topics in daily work

---

### Section 7: Common Workflows
**Purpose:** Step-by-step guides for frequent tasks

```markdown
## Common Workflows

### Rebuild After Code Changes
```bash
# 1. Clean build (if headers changed)
./waf clean

# 2. Reconfigure (only if build system changed)
./waf configure --board sitl --enable-dds

# 3. Build
./waf copter

# 4. Verify binary
ls -lh build/sitl/bin/arducopter
```

### Test DDS Communication
```bash
# 1. Start Micro-ROS Agent
ros2 run micro_ros_agent micro_ros_agent udp4 --port {2019+(N-1)*10} --namespace ap_{N}

# 2. In another terminal, start SITL
build/sitl/bin/arducopter --model json --speedup 1 --instance {N} \
  --defaults Tools/ros2/ardupilot_sitl/config/default_params/dds_udp_instance{N}.parm

# 3. Verify topics
ros2 topic list | grep /ap_{N}/
ros2 topic echo /ap_{N}/pose/filtered --once
```

### Add a New DDS Topic
1. Edit `libraries/AP_DDS/AP_DDS_Topic_Table.h`
2. Add topic entry to table
3. Implement publisher in `libraries/AP_DDS/AP_DDS_Client.cpp`
4. Rebuild: `./waf copter`
5. Test with `ros2 topic echo`

### Debug Gazebo Connection Issues
```bash
# Check if FDM ports are open
netstat -an | grep {9002+(N-1)*10}

# Monitor JSON packets (requires tcpdump)
sudo tcpdump -i lo -A port {9002+(N-1)*10}

# Check ArduPilot logs
tail -f /tmp/ArduCopter.log
```
```

**Content:**
- Rebuild workflow
- Test DDS in isolation
- Add new ROS2 topic
- Debug Gazebo connection
- Debug DDS connection
- Change parameters

**Rationale:** Workflows save time and prevent errors

---

### Section 8: Troubleshooting
**Purpose:** Common problems and solutions

```markdown
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

# Verify Micro-XRCE-DDS-Client exists
ls modules/Micro-XRCE-DDS-Client/
```

---

**Problem:** Compilation errors in `AP_DDS_Client.cpp`
```
error: 'dds_write' was not declared in this scope
```

**Solution:**
```bash
# Clean and rebuild DDS components
./waf clean
./waf configure --board sitl --enable-dds
./waf copter
```

---

### Runtime Issues

**Problem:** No ROS2 topics appearing
```
$ ros2 topic list | grep /ap_{N}/
(empty)
```

**Checklist:**
1. Is Micro-ROS Agent running? `ps aux | grep micro_ros_agent`
2. Is DDS port correct in parm file? `grep DDS_UDP_PORT Tools/ros2/ardupilot_sitl/config/default_params/dds_udp_instance{N}.parm`
3. Is ArduPilot SITL running? `ps aux | grep arducopter`
4. Check ArduPilot startup output for DDS errors

---

**Problem:** SITL won't connect to Gazebo
```
Failed to connect to Gazebo on port {9002+(N-1)*10}
```

**Checklist:**
1. Is Gazebo running? `gz sim --list`
2. Is ArduPilotPlugin loaded? `gz model --list | grep iris_{9002+(N-1)*10}`
3. Is FDM port correct in model.sdf?
4. Check Gazebo logs: `gz log`

---

### Port Conflicts

**Problem:** "Address already in use" error

**Solution:**
```bash
# Find process using port
lsof -i :{PORT}

# Kill old SITL process if stuck
pkill arducopter

# Verify ports are free
netstat -an | grep {2019+(N-1)*10}
netstat -an | grep {9002+(N-1)*10}
```
```

**Content:**
- Build failures (submodules, dependencies)
- Runtime issues (no topics, no Gazebo connection)
- Port conflicts
- Common error messages with solutions
- Diagnostic commands

**Rationale:** Troubleshooting saves hours of debugging

---

### Section 9: Development Guidelines
**Purpose:** Code style and contribution guidelines

```markdown
## Development Guidelines

### Code Style
- Follow ArduPilot coding standards
- Use 4-space indentation (not tabs)
- Max line length: 120 characters
- Always run `./waf check` before committing

### Testing
- Test SITL standalone before multi-drone launch
- Verify ROS2 topics after DDS changes
- Check MAVLink compatibility with MAVProxy

### Making Changes
1. **Branch Strategy**: Create feature branch from master
2. **Build Verification**: `./waf copter` must succeed
3. **Integration Test**: Test with workspace launch file
4. **Documentation**: Update CLAUDE.md if ports/config change

### Debugging Tools
- **GDB:** `gdb --args build/sitl/bin/arducopter ...`
- **Logs:** `/tmp/ArduCopter.log`
- **MAVLink:** MAVProxy console for real-time telemetry
- **ROS2:** `ros2 topic echo` for DDS verification
```

**Content:**
- Code formatting rules
- Testing requirements
- Development workflow
- Debugging tools
- When to update documentation

**Rationale:** Sets expectations for AI-assisted development

---

### Section 10: Links and References
**Purpose:** Pointers to additional documentation

```markdown
## References

### Project Documentation
- **Workspace CLAUDE.md:** `/home/finn/Documents/ardu_ws/.claude/CLAUDE.md` (system architecture)
- **RESEARCH.md:** `.claude/10092025 update claude docs/RESEARCH.md` (detailed technical analysis)
- **Multi-Drone Launch:** `ardupilot_gz_bringup/launch/iris_multi_uav.launch.py`

### Upstream Documentation
- **ArduPilot SITL:** https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html
- **ArduPilot DDS:** https://ardupilot.org/dev/docs/ros2.html
- **Gazebo ArduPilot Plugin:** https://github.com/ArduPilot/ardupilot_gazebo
- **Micro-XRCE-DDS:** https://micro-xrce-dds.docs.eprosima.com/

### Related Repositories
- **ardupilot_gazebo:** Gazebo models and plugins
- **ardupilot_gz_bringup:** ROS2 launch files
- **formation_control:** High-level formation control nodes
```

**Content:**
- Internal project docs
- Upstream ArduPilot docs
- Related component repos
- External dependency docs

**Rationale:** Quick access to authoritative sources

---

## What NOT to Include in CLAUDE.md

To keep the file concise and focused:

### ❌ Exclude: Deep Technical Theory
- EKF mathematics and algorithms (in upstream docs)
- PID controller tuning theory (ArduPilot docs)
- DDS/RTPS protocol specifications (eProsima docs)
- JSON/UDP packet format details (see RESEARCH.md)

**Rationale:** Link to sources; don't duplicate

### ❌ Exclude: Multi-Instance Architecture
- System-level architecture diagrams (in workspace CLAUDE.md)
- Complete 5-instance port matrix (in RESEARCH.md)
- Launch sequence choreography (in workspace docs)
- Formation control algorithms (in formation_control package)

**Rationale:** That's workspace-level context, not instance-level

### ❌ Exclude: Historical Information
- Why we chose ArduPilot over PX4
- Git commit history and decisions
- Previous architecture iterations

**Rationale:** Not actionable for current work

### ❌ Exclude: Gazebo Plugin Internals
- ArduPilotPlugin source code details
- Lock-step synchronization algorithms
- Motor physics models

**Rationale:** That's in ardupilot_gazebo repo, not here

### ❌ Exclude: Complete ROS2 Message Specs
- Full IDL file contents
- Every field of every message type
- QoS policy deep dives

**Rationale:** Use `ros2 interface show` instead

---

## Instance-Specific Customization

The CLAUDE.md must be **customized for each instance**:

### Template Variables to Replace

| Placeholder | Instance 1 | Instance 2 | Instance 3 | Instance 4 | Instance 5 |
|-------------|-----------|-----------|-----------|-----------|-----------|
| `{N}` | 1 | 2 | 3 | 4 | 5 |
| `{DDS_PORT}` | 2019 | 2029 | 2039 | 2049 | 2059 |
| `{FDM_IN}` | 9002 | 9012 | 9022 | 9032 | 9042 |
| `{FDM_OUT}` | 9003 | 9013 | 9023 | 9033 | 9043 |
| `{TCP_MAV}` | 5760 | 5770 | 5780 | 5790 | 5800 |
| `{UDP_MAV}` | 14550 | 14560 | 14570 | 14580 | 14590 |
| `{SITL_PORT}` | 5501 | 5511 | 5521 | 5531 | 5541 |
| `{GAZEBO_MODEL}` | iris_with_gimbal_9002 | iris_with_gimbal_9012 | iris_with_gimbal_9022 | iris_with_gimbal_9032 | iris_with_gimbal_9042 |

### Implementation Strategy

**Option A: Separate Files (Recommended)**
- Create `CLAUDE.md` in each `ardupilotN/` directory
- Replace all placeholders with actual values
- Each file is standalone and correct

**Option B: Templating System**
- Create `CLAUDE.template.md` with placeholders
- Use script to generate 5 versions
- Risk: Placeholders might confuse Claude

**Recommendation:** Option A - Be explicit, not clever

---

## Validation Checklist

Before finalizing CLAUDE.md, verify:

### Content Completeness
- [ ] Instance number is clearly stated
- [ ] All port numbers are correct for this instance
- [ ] File paths are absolute and correct
- [ ] Commands are copy-pastable
- [ ] Links to RESEARCH.md and workspace CLAUDE.md work

### Accuracy
- [ ] Build commands tested and working
- [ ] Port calculations verified
- [ ] ROS2 namespace matches launch file
- [ ] File paths exist on filesystem

### Usability
- [ ] Table of contents if >300 lines
- [ ] Code blocks have language hints (```bash)
- [ ] No broken markdown formatting
- [ ] Consistent heading levels

### Maintainability
- [ ] Last updated date included
- [ ] Version number or commit hash
- [ ] Clear ownership/author attribution
- [ ] Instructions for updating CLAUDE.md

---

## Implementation Recommendations

### Step 1: Create Template
1. Write `CLAUDE.template.md` following structure above
2. Use `{N}`, `{DDS_PORT}`, etc. placeholders
3. Include all 10 sections

### Step 2: Generate Instance Files
```bash
for i in 1 2 3 4 5; do
  sed -e "s/{N}/$i/g" \
      -e "s/{DDS_PORT}/$((2019 + (i-1)*10))/g" \
      -e "s/{FDM_IN}/$((9002 + (i-1)*10))/g" \
      -e "s/{FDM_OUT}/$((9003 + (i-1)*10))/g" \
      -e "s/{TCP_MAV}/$((5760 + (i-1)*10))/g" \
      -e "s/{UDP_MAV}/$((14550 + (i-1)*10))/g" \
      -e "s/{SITL_PORT}/$((5501 + (i-1)*10))/g" \
      -e "s/{GAZEBO_MODEL}/iris_with_gimbal_$((9002 + (i-1)*10))/g" \
      CLAUDE.template.md > ../ardupilot$i/.claude/CLAUDE.md
done
```

### Step 3: Manual Review
- Check each generated file
- Verify port calculations
- Test commands
- Validate links

### Step 4: Version Control
```bash
cd /home/finn/Documents/ardu_ws/src
git add ardupilot*/.claude/CLAUDE.md
git commit -m "docs: Add instance-specific CLAUDE.md files"
```

---

## Success Metrics

The CLAUDE.md is successful if Claude can:

1. **Identify Context:** "Which instance am I working in?"
2. **Build Correctly:** Rebuild after code changes
3. **Test Changes:** Verify ROS2 integration works
4. **Debug Issues:** Diagnose common problems
5. **Navigate Code:** Find relevant files quickly
6. **Understand Integration:** Know how this instance fits in system

**Test:** Give Claude a task like "Add a new ROS2 topic that publishes wind speed at 10 Hz" and see if CLAUDE.md provides sufficient context to complete it correctly for the specific instance.

---

## Maintenance Plan

### When to Update CLAUDE.md

1. **Port Changes:** If instance port assignments change
2. **New Features:** When adding ROS2 topics/services
3. **Build Process Changes:** If waf commands change
4. **File Reorganization:** If key files move
5. **Integration Changes:** If launch process changes

### Review Cadence
- **Monthly:** Quick review for accuracy
- **Major Updates:** When ArduPilot version bumps
- **On Issues:** If Claude struggles with a task, add to troubleshooting

### Ownership
- Primary: Project maintainer
- Contributors: Anyone who finds inaccuracies
- Version control: Git tracks all changes

---

## Appendix: Best Practices Applied

This plan applies Claude Code best practices as follows:

| Best Practice | How Applied |
|---------------|-------------|
| **Be Specific** | Exact port numbers, complete commands, absolute paths |
| **Organizational Structure** | 10 clear sections with headings and tables |
| **Actionable Content** | Quick reference, workflows, troubleshooting |
| **Conciseness** | 200-400 lines vs 838 in RESEARCH.md |
| **Periodic Review** | Maintenance plan with review cadence |
| **Hierarchical Approach** | Instance CLAUDE.md → Workspace CLAUDE.md → RESEARCH.md |

---

## Next Steps

1. **Review This Plan:** Get feedback on structure and content
2. **Create Template:** Write `CLAUDE.template.md` with all sections
3. **Generate Instances:** Use sed script to create 5 versions
4. **Test with Claude:** Have Claude attempt common tasks using only CLAUDE.md
5. **Iterate:** Refine based on what Claude struggles with
6. **Deploy:** Commit to version control
7. **Document Process:** Add "How to update CLAUDE.md" to workspace docs

---

**Plan Version:** 1.0
**Author:** Claude Code (Anthropic)
**Date:** 2025-10-09
**Status:** Ready for Implementation
