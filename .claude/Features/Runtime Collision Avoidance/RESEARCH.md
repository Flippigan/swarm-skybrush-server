# Runtime Collision Avoidance in Skybrush Server
## Research Report

**Date**: 2025-10-31
**Analyzed Component**: Geofence Collision Avoidance (`src/flockwave/server/model/geofence.py:60-66`)
**Skybrush Server Version**: 2.37.0

---

## Executive Summary

This report analyzes the collision avoidance implementation in Skybrush Server, specifically the `SMART_RETURN` and `SMART_LAND` geofence actions. **Key finding**: The Skybrush server does **NOT** implement collision avoidance logic itself. Instead, it acts as a configuration layer that delegates all collision avoidance to individual autopilot firmware (ArduPilot).

**Critical Gaps Identified**:
- No server-side trajectory collision checking
- No inter-drone proximity detection
- No centralized swarm-level collision avoidance
- No runtime trajectory validation

---

## 1. Overview: SMART Geofence Actions

### Definition

Located in `src/flockwave/server/model/geofence.py` (lines 53-76):

```python
class GeofenceAction(Enum):
    """Actions that a UAV can take when hitting the geofence."""

    REPORT = "report"
    """Report the geofence violation but do nothing."""

    SMART_RETURN = "smartReturn"
    """Attempt to return to the launch site with smart collision avoidance."""

    RETURN = "return"
    """Attempt to return to the launch site without collision avoidance."""

    SMART_LAND = "smartLand"
    """Attempt to land in-place with collision avoidance."""

    LAND = "land"
    """Attempt to land in-place without collision avoidance."""

    STOP = "stop"
    """Stop and hover in-place."""

    SHUT_DOWN = "shutDown"
    """Shut down immediately."""
```

### Purpose

These actions define what a drone should do when it violates a configured geofence (altitude limits, distance from home, or geographic boundaries).

---

## 2. Implementation Architecture

### Server vs Autopilot Responsibilities

| Component | Responsibility | Location |
|-----------|---------------|----------|
| **Skybrush Server** | Parse geofence configs, translate to parameters, upload to drones | `src/flockwave/server/` |
| **ArduPilot Firmware** | Monitor fences, detect violations, execute actions, perform collision avoidance | Runs on flight controller |

### Data Flow

```
Show File (.skyb with geofence spec)
    ↓
src/flockwave/server/show/geofence.py (parse geofence from show)
    ↓
src/flockwave/server/model/geofence.py (GeofenceAction data structures)
    ↓
src/flockwave/server/ext/mavlink/autopilots/ardupilot.py (translate to FENCE_ACTION)
    ↓
src/flockwave/server/ext/mavlink/geofence.py (upload via MAVLink mission protocol)
    ↓
ArduPilot Firmware (FENCE_ACTION parameter + Smart RTL mode)
    ↓
[ACTUAL COLLISION AVOIDANCE HAPPENS HERE IN FIRMWARE]
```

---

## 3. ArduPilot Integration Details

### Geofence Action Mapping

From `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` (lines 92-98):

```python
_geofence_actions = {
    0: (GeofenceAction.REPORT,),
    1: (GeofenceAction.RETURN, GeofenceAction.LAND),
    2: (GeofenceAction.LAND,),
    3: (GeofenceAction.SMART_RETURN, GeofenceAction.RETURN, GeofenceAction.LAND),
    4: (GeofenceAction.STOP, GeofenceAction.LAND),
}
```

**Interpretation**:
- ArduPilot `FENCE_ACTION=0` → Report only
- ArduPilot `FENCE_ACTION=1` → RTL or Land (fallback chain)
- ArduPilot `FENCE_ACTION=2` → Land immediately
- ArduPilot `FENCE_ACTION=3` → **Smart RTL → RTL → Land** (with collision avoidance)
- ArduPilot `FENCE_ACTION=4` → Stop (brake) or Land

### Reading Geofence Status

From `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` (lines 449-472):

```python
async def get_geofence_status(self, uav: MAVLinkUAV) -> GeofenceStatus:
    """Returns the status of the geofence on the given MAVLink UAV."""
    status = GeofenceStatus()

    # Get fence polygons, circles, and rally points
    manager = GeofenceManager.for_uav(uav)
    await manager.get_geofence_areas_and_rally_points(status)

    # Get fence enable status
    value = await uav.get_parameter("FENCE_ENABLE")
    status.enabled = bool(value)

    # Get fence action and translate to high-level actions
    value = await uav.get_parameter("FENCE_ACTION")
    status.actions = list(self._geofence_actions.get(int(value), ()))

    # Get fence altitude and distance parameters
    # [additional code for min/max altitude, max distance...]

    return status
```

---

## 4. Critical Limitation: SMART Actions Not Directly Configurable

### The Problem

From `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` (lines 342-356):

```python
if configuration.action is not None:
    # Update geofence action
    action_map = {
        GeofenceAction.LAND: 2,      # always land
        GeofenceAction.REPORT: 0,    # report only
        GeofenceAction.RETURN: 1,    # RTH or land
        GeofenceAction.STOP: 4,      # brake or land
    }
    mapped_action = action_map.get(configuration.action)
    if mapped_action is not None:
        await uav.set_parameter("FENCE_ACTION", int(mapped_action))
    else:
        raise NotSupportedError(
            f"geofence action {configuration.action!r} not supported on ArduPilot"
        )
```

**Key Issue**: `SMART_RETURN` and `SMART_LAND` are **NOT in the action_map**.

**Consequence**: You cannot directly configure a drone to use `SMART_RETURN` or `SMART_LAND` via the Skybrush server API. If you try, you'll get a `NotSupportedError`.

**Why They Exist**: These actions appear in the enum because ArduPilot `FENCE_ACTION=3` is *interpreted* as a fallback chain that includes smart return, but you cannot set `FENCE_ACTION=3` by requesting `SMART_RETURN`.

---

## 5. How ArduPilot Smart RTL Actually Works

### Flight Mode 21: Smart RTL

From `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` (line 82):

```python
_flight_modes = {
    # ... other modes ...
    21: "SMART_RTL",
    # ... other modes ...
}
```

### Smart RTL Features (ArduPilot Firmware)

1. **Path Recording**: During normal flight, ArduPilot records the vehicle's path
2. **Intelligent Return**: When Smart RTL is triggered, the vehicle retraces its path in reverse
3. **Path Simplification**: The recorded path is simplified to reduce waypoints
4. **Safe Return**: Avoids obstacles by following known-safe paths

### Object Avoidance Integration

ArduPilot supports object avoidance through proximity sensors and path planning:

**OA Parameters** (configurable in ArduPilot):
- `OA_TYPE`: Object avoidance type (0=disabled, 1=BendyRuler, 2=Dijkstra)
- `OA_DB_SIZE`: Database size for path planning
- `OA_MARGIN_MAX`: Maximum margin around obstacles

**When Enabled**:
- Uses Dijkstra's algorithm for path planning around fences
- Works in Auto, Guided, and RTL modes
- Requires proximity sensors or fence definitions

### Backup Fence Mechanism

From ArduPilot documentation:
1. When primary fence is breached, a backup fence is erected 20m further out
2. If backup fence is also breached, `FENCE_ACTION` executes again
3. If vehicle goes 100m outside configured fence, switches to LAND mode regardless

---

## 6. Server-Side Code Analysis: What's Missing

### Areas Searched

I extensively searched the following modules for collision detection logic:

| Module | File | Collision Detection Found? |
|--------|------|---------------------------|
| Show Player | `src/flockwave/server/show/player.py` | ❌ None |
| Trajectory | `src/flockwave/server/show/trajectory.py` | ❌ None |
| Safety | `src/flockwave/server/show/safety.py` | ❌ None (only battery thresholds) |
| Geofence Processing | `src/flockwave/server/show/geofence.py` | ❌ None (only parsing) |
| Mission Execution | `src/flockwave/server/ext/missions/` | ❌ None |
| Virtual UAVs | `src/flockwave/server/ext/virtual_uavs/` | ⚠️ Only static placement spacing |

### Virtual UAV Placement (Not Runtime Collision Avoidance)

From `src/flockwave/server/ext/virtual_uavs/placement.py` (lines 106-139):

```python
@register("circle")
def place_drones_on_circle(
    n: int, *, radius: Optional[float] = None, min_distance: float = 5
) -> list[Vector3D]:
    """Returns coordinates to place the given number of drones in a circle.

    Parameters:
        n: the number of drones to place
        radius: the radius of the circle
        min_distance: the minimum distance between drones in the circle, along
            the circumference of the circle. Used only when the radius is not
            given explicitly.
    """
    if radius is None:
        # Calculate radius based on minimum distance
        radius = n * min_distance / 2 / pi

    # Generate coordinates...
```

**Note**: This is only for **initial placement** of virtual drones, not for runtime collision avoidance during flight.

---

## 7. Safety Configuration Scope

### What Safety Configuration Covers

From `src/flockwave/server/model/safety.py` (lines 50-66):

```python
@dataclass
class SafetyConfigurationRequest:
    """Object representing a safety configuration object that can be enforced
    on a drone.

    This is admittedly minimal for the time being. We can update it as we
    implement support for more complex safety features. Things that are missing:

    - all kinds of failsafe settings
    - detailed proximity sensing setup (usage, distance from objects etc.)
    - threshold deviation from waypoint line that triggers an error
    - etc.
    """

    low_battery_threshold: Optional[LowBatteryThreshold] = None
    critical_battery_voltage: Optional[float] = None
    return_to_home_altitude: Optional[float] = None
    return_to_home_speed: Optional[float] = None
```

**Commentary**: The developers explicitly acknowledge that proximity sensing and collision avoidance are **NOT implemented**.

---

## 8. Testing Coverage

### Search Results

**Test Files Examined**:
- `test/test_ext_mavlink_autopilots_ardupilot.py`
- `test/test_ext_mavlink_packets.py`
- `test/test_ext_show_*.py`
- `test/test_preflight_check.py`

**Findings**:
- Only one test references `FENCE_ACTION` (line 31 of `test_ext_mavlink_autopilots_ardupilot.py`)
- Test simply validates parameter encoding, not collision avoidance behavior
- **No tests for collision detection or avoidance**

**Conclusion**: Collision avoidance is considered autopilot firmware functionality, not something the server tests.

---

## 9. Complete Architecture Flow

### Geofence Violation Handling

```
┌─────────────────────────────────────────────────────────────────┐
│                     SKYBRUSH SERVER                              │
│                                                                  │
│  1. Parse show file with geofence specification                 │
│  2. Create GeofenceConfigurationRequest object                  │
│  3. Translate to autopilot-specific parameters:                 │
│     - FENCE_ENABLE (0/1)                                        │
│     - FENCE_ACTION (0-4)                                        │
│     - FENCE_ALT_MIN, FENCE_ALT_MAX                              │
│     - FENCE_RADIUS                                              │
│     - Fence polygons/circles via mission items                  │
│  4. Upload to each drone via MAVLink                            │
│  5. Monitor fence status (read parameters back)                 │
│                                                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │ MAVLink Protocol
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                  ARDUPILOT FIRMWARE (on each drone)              │
│                                                                  │
│  1. Continuously monitor position vs fence boundaries           │
│  2. When fence breached:                                        │
│     a. Set FENCE_BREACHED flag                                  │
│     b. Execute action based on FENCE_ACTION:                    │
│                                                                  │
│        FENCE_ACTION=0: Log violation only                       │
│        FENCE_ACTION=1: Switch to RTL mode → Land                │
│        FENCE_ACTION=2: Switch to LAND mode immediately          │
│        FENCE_ACTION=3: Switch to SMART_RTL → RTL → Land        │
│        FENCE_ACTION=4: Switch to BRAKE → Land                   │
│                                                                  │
│  3. If Smart RTL selected:                                      │
│     a. Load recorded path from memory                           │
│     b. Reverse and simplify path                                │
│     c. If OA enabled: Use Dijkstra for obstacle avoidance       │
│     d. Follow path back to home                                 │
│     e. Land at home position                                    │
│                                                                  │
│  4. Backup fence mechanism:                                     │
│     - Create backup fence 20m beyond primary                    │
│     - If backup breached: re-execute FENCE_ACTION               │
│     - If >100m out: Force LAND mode                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

[COLLISION AVOIDANCE HAPPENS HERE IN ARDUPILOT FIRMWARE]
```

---

## 10. Recommendations for True Swarm Collision Avoidance

The current implementation does **NOT** provide swarm-level collision avoidance. To implement this, you would need:

### Option 1: Server-Side Pre-Flight Validation

**Location**: Enhance `src/flockwave/server/show/player.py`

**Implementation**:
1. Before uploading show trajectories, validate all drone paths
2. Check for potential collisions between drones
3. Verify minimum separation distances at all time points
4. Reject shows that have collision risks

**Advantages**:
- Catches problems before flight
- Works with existing hardware

**Limitations**:
- No runtime protection
- Can't handle GPS drift or positioning errors

### Option 2: Runtime Proximity Monitoring

**Location**: New module `src/flockwave/server/ext/collision_monitor/`

**Implementation**:
1. Subscribe to position updates from all drones
2. Calculate inter-drone distances in real-time
3. Trigger warnings when drones get too close
4. Send emergency stop/hover commands if collision imminent

**Advantages**:
- Real-time protection
- Can handle GPS drift

**Limitations**:
- Requires fast communication
- May have false positives

### Option 3: Distributed Collision Avoidance (ROS2 Layer)

**Location**: Implement in your ROS2 formation control workspace

**Implementation**:
1. Run collision avoidance in formation control nodes
2. Adjust formation commands before sending to ArduPilot
3. Use velocity-based obstacle avoidance (ORCA, RVO)
4. Maintain minimum separation via potential fields

**Advantages**:
- Best for research and custom formations
- Full control over algorithms
- Can integrate with your existing ROS2 architecture

**Limitations**:
- More complex to implement
- Requires tight ROS2-ArduPilot integration

---

## 11. File Reference Guide

### Core Geofence Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/flockwave/server/model/geofence.py` | 1-248 | GeofenceAction enum, data structures |
| `src/flockwave/server/ext/mavlink/geofence.py` | Full | MAVLink geofence upload/download protocol |
| `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` | 92-98 | Action mapping dictionary |
| `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` | 342-356 | Set geofence action (with limitation) |
| `src/flockwave/server/ext/mavlink/autopilots/ardupilot.py` | 449-472 | Get geofence status |
| `src/flockwave/server/show/geofence.py` | Full | Parse geofence from show files |

### Safety and Configuration

| File | Lines | Purpose |
|------|-------|---------|
| `src/flockwave/server/model/safety.py` | 50-113 | SafetyConfigurationRequest (no collision avoidance) |
| `src/flockwave/server/show/safety.py` | Full | Parse safety config from show files |
| `src/flockwave/server/ext/mavlink/autopilots/base.py` | 151-154 | Documentation on NotSupportedError |

### Placement (Not Collision Avoidance)

| File | Lines | Purpose |
|------|-------|---------|
| `src/flockwave/server/ext/virtual_uavs/placement.py` | 106-139 | Circle placement with min_distance |
| `src/flockwave/server/ext/virtual_uavs/placement.py` | 142-191 | Grid placement with spacing |

---

## 12. Conclusion

### Summary of Findings

1. **No Server-Side Collision Avoidance**: The Skybrush server does not implement any collision detection or avoidance logic

2. **Autopilot-Delegated**: All collision avoidance is handled by individual ArduPilot firmware instances using:
   - Smart RTL mode (flight mode 21)
   - Object Avoidance with Dijkstra path planning
   - Proximity sensors (if hardware available)

3. **Geofence Actions**: `SMART_RETURN` and `SMART_LAND` are high-level descriptors that map to ArduPilot's `FENCE_ACTION=3`, but cannot be directly configured

4. **Configuration Limitation**: The action_map explicitly excludes `SMART_RETURN` and `SMART_LAND`, raising `NotSupportedError` if you try to set them

5. **Architecture Philosophy**: Skybrush follows a **coordinated choreography** model:
   - Trajectories are pre-planned and validated offline (in Skybrush Studio)
   - Server executes pre-validated trajectories
   - Individual autopilots handle real-time safety

6. **Gap for Swarm Operations**: For true multi-drone collision avoidance, you need either:
   - Pre-flight trajectory validation
   - Runtime proximity monitoring with emergency stop
   - External collision avoidance layer (e.g., ROS2)

### For Your Use Case

Based on your ROS2 workspace context (from the CLAUDE.md file showing formation control, safety monitor, etc.), you should implement collision avoidance in **your ROS2 formation control layer**, not by extending Skybrush Server.

The Skybrush server is designed for executing pre-planned drone shows, not for dynamic swarm control with reactive collision avoidance.

---

**End of Report**
