# Dreame L10 Pro - Valetudo Communication API Documentation

## Table of Contents
- [Overview](#overview)
- [Communication Architecture](#communication-architecture)
- [MIoT Protocol Structure](#miot-protocol-structure)
- [Command Examples](#command-examples)
- [Complete MIoT Service Reference](#complete-miot-service-reference)
- [Status and Error Codes](#status-and-error-codes)
- [Map Data Flow](#map-data-flow)

---

## Overview

The Dreame L10 Pro vacuum robot communicates using the **MIoT (Xiaomi IoT) Protocol** over the Miio transport layer. Valetudo intercepts and controls these communications locally.

**Your Robot Model**: `dreame.vacuum.p2029` (Dreame L10 Pro)
**MIoT Specification**: GEN2 (Generation 2)
**Protocol**: UDP-based encrypted communication on port 8053

---

## Communication Architecture

### Transport Layers

1. **Local Socket** (Primary)
   - UDP socket bound to `127.0.0.1:8053`
   - Encrypted using device token from `/data/config/miio/device.token`
   - Direct communication with robot firmware

2. **Dummycloud** (Cloud Interception)
   - Spoofed Xiaomi cloud server running on `127.0.0.1:8053`
   - Intercepts cloud-bound traffic from robot
   - Used for map data transfers and push notifications

3. **FDS Mock Server**
   - HTTP server on `127.0.0.1:8079`
   - Receives map data uploads from robot
   - Endpoint: `PUT /api/miio/fds_upload_handler`

### Communication Flow

```
┌─────────────────┐
│  Valetudo UI    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│ DreameMiotHelper│◄─────┤ MiioValetudoRobot│
└────────┬────────┘      └──────────────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│  sendCommand()  │◄─────┤   Local Socket   │
└────────┬────────┘      │   (UDP:8053)     │
         │               └──────────────────┘
         ▼
┌─────────────────┐
│ Robot Firmware  │
└─────────────────┘
```

---

## MIoT Protocol Structure

### Core Concepts

MIoT uses a hierarchical addressing system:

- **DID** (Device ID): Unique identifier for the robot
- **SIID** (Service Instance ID): Identifies a service/subsystem
- **PIID** (Property Instance ID): Identifies a property within a service
- **AIID** (Action Instance ID): Identifies an action within a service

### Three Operation Types

#### 1. Read Property (`get_properties`)
Retrieve current values from the robot.

**Request Format**:
```json
{
  "method": "get_properties",
  "params": [
    {
      "did": <device_id>,
      "siid": <service_id>,
      "piid": <property_id>
    }
  ]
}
```

**Response Format**:
```json
[
  {
    "code": 0,
    "siid": <service_id>,
    "piid": <property_id>,
    "value": <property_value>
  }
]
```

#### 2. Write Property (`set_properties`)
Set values on the robot.

**Request Format**:
```json
{
  "method": "set_properties",
  "params": [
    {
      "did": <device_id>,
      "siid": <service_id>,
      "piid": <property_id>,
      "value": <new_value>
    }
  ]
}
```

**Response Format**:
```json
[
  {
    "code": 0,
    "siid": <service_id>,
    "piid": <property_id>
  }
]
```

#### 3. Execute Action (`action`)
Trigger actions on the robot.

**Request Format**:
```json
{
  "method": "action",
  "params": {
    "did": <device_id>,
    "siid": <service_id>,
    "aiid": <action_id>,
    "in": [
      {
        "piid": <input_property_id>,
        "value": <input_value>
      }
    ]
  }
}
```

**Response Format**:
```json
{
  "code": 0,
  "out": [
    {
      "piid": <output_property_id>,
      "value": <output_value>
    }
  ]
}
```

**Response Codes**:
- `0` = Success
- `1` = Invalid parameters
- `-4001` = Property not supported
- `-4002` = Value out of range

---

## Command Examples

### Example 1: Start Cleaning
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 2,
    "aiid": 1
  }
}
```
**Translation**: VACUUM_1.RESUME → Start/Resume cleaning

---

### Example 2: Set Fan Speed to High
```json
{
  "method": "set_properties",
  "params": [
    {
      "did": 277962183,
      "siid": 4,
      "piid": 4,
      "value": 2
    }
  ]
}
```
**Translation**: VACUUM_2.FAN_SPEED = 2 (High)

**Fan Speed Values**:
- `0` = Low
- `1` = Medium
- `2` = High
- `3` = Max/Turbo

---

### Example 3: Return to Dock
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 3,
    "aiid": 1
  }
}
```
**Translation**: BATTERY.START_CHARGE → Return to charging dock

---

### Example 4: Read Battery Level
```json
{
  "method": "get_properties",
  "params": [
    {
      "did": 277962183,
      "siid": 3,
      "piid": 1
    }
  ]
}
```

**Response**:
```json
[
  {
    "code": 0,
    "siid": 3,
    "piid": 1,
    "value": 85
  }
]
```
**Translation**: BATTERY.LEVEL = 85%

---

### Example 5: Poll Multiple Properties (Robot State)
```json
{
  "method": "get_properties",
  "params": [
    {"did": 277962183, "siid": 4, "piid": 1},
    {"did": 277962183, "siid": 4, "piid": 7},
    {"did": 277962183, "siid": 4, "piid": 4},
    {"did": 277962183, "siid": 4, "piid": 18},
    {"did": 277962183, "siid": 3, "piid": 1},
    {"did": 277962183, "siid": 3, "piid": 2}
  ]
}
```

**Response**:
```json
[
  {"code": 0, "siid": 4, "piid": 1, "value": 0},
  {"code": 0, "siid": 4, "piid": 7, "value": 0},
  {"code": 0, "siid": 4, "piid": 4, "value": 2},
  {"code": 0, "siid": 4, "piid": 18, "value": 0},
  {"code": 0, "siid": 3, "piid": 1, "value": 85},
  {"code": 0, "siid": 3, "piid": 2, "value": 1}
]
```

**Translation**:
- MODE = 0 (Idle)
- TASK_STATUS = 0 (No pending task)
- FAN_SPEED = 2 (High)
- ERROR_CODE = 0 (No error)
- BATTERY.LEVEL = 85%
- BATTERY.CHARGING = 1 (On charger)

---

### Example 6: Request Map Data
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 6,
    "aiid": 1,
    "in": [
      {
        "piid": 2,
        "value": "{\"frame_type\":\"I\", \"force_type\": 1, \"req_type\": 1}"
      }
    ]
  }
}
```

**Response**:
```json
{
  "code": 0,
  "out": [
    {
      "piid": 1,
      "value": "H4sIAAAAAAAA/+2dZ3Mc..."
    }
  ]
}
```
**Translation**: MAP.POLL → Returns base64-encoded, zlib-compressed map data

**Frame Types**:
- `I` = I-Frame (full map)
- `P` = P-Frame (partial update)

---

### Example 7: Set Water Usage to Medium
```json
{
  "method": "set_properties",
  "params": [
    {
      "did": 277962183,
      "siid": 4,
      "piid": 5,
      "value": 2
    }
  ]
}
```
**Translation**: VACUUM_2.WATER_USAGE = 2 (Medium)

**Water Usage Values**:
- `1` = Low
- `2` = Medium
- `3` = High

---

### Example 8: Play "Find Me" Sound
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 7,
    "aiid": 1
  }
}
```
**Translation**: AUDIO.LOCATE → Play location sound

---

### Example 9: Reset Main Brush Consumable Timer
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 9,
    "aiid": 1
  }
}
```
**Translation**: MAIN_BRUSH.RESET → Reset main brush lifetime counter

---

### Example 10: Enable Do Not Disturb
```json
{
  "method": "set_properties",
  "params": [
    {
      "did": 277962183,
      "siid": 5,
      "piid": 1,
      "value": 1
    },
    {
      "did": 277962183,
      "siid": 5,
      "piid": 2,
      "value": "22:00"
    },
    {
      "did": 277962183,
      "siid": 5,
      "piid": 3,
      "value": "08:00"
    }
  ]
}
```
**Translation**:
- DND.ENABLED = 1 (On)
- DND.START_TIME = "22:00"
- DND.END_TIME = "08:00"

---

## Complete MIoT Service Reference

### DEVICE (SIID: 1)
General device information.

| Property | PIID | Type | Description |
|----------|------|------|-------------|
| SERIAL_NUMBER | 5 | string | Device serial number |

---

### VACUUM_1 (SIID: 2)
Primary vacuum status service.

#### Properties
| Property | PIID | Type | Values | Description |
|----------|------|------|--------|-------------|
| STATUS | 1 | int | 0-23 | Extended status enum (newer robots) |
| ERROR | 2 | int | 0-121 | Legacy error code |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESUME | 1 | Start or resume cleaning |
| PAUSE | 2 | Pause cleaning |

---

### VACUUM_2 (SIID: 4)
Extended vacuum control and settings.

#### Properties
| Property | PIID | Type | Values | Description |
|----------|------|------|--------|-------------|
| MODE | 1 | int | 0-23 | Current operation mode (see [Status Codes](#mode-values)) |
| CLEANING_TIME | 2 | int | seconds | Current cleaning session duration |
| CLEANING_AREA | 3 | int | cm² | Current cleaning session area |
| FAN_SPEED | 4 | int | 0-3 | 0=Low, 1=Med, 2=High, 3=Max |
| WATER_USAGE | 5 | int | 1-3 | 1=Low, 2=Med, 3=High |
| WATER_TANK_ATTACHMENT | 6 | int | 0-3 | Bitmask: 0=None, 1=Tank, 2=Mop, 3=Both |
| TASK_STATUS | 7 | int | 0/1 | 0=Has pending task, 1=No task |
| STATE_CHANGE_TIMESTAMP | 8 | int | unix | Last state change timestamp |
| ADDITIONAL_CLEANUP_PROPERTIES | 10 | string | JSON | Zone/segment cleaning parameters |
| POST_CHARGE_CONTINUE | 11 | bool | 0/1 | Resume cleaning after charging |
| CARPET_MODE | 12 | int | 0-2 | 0=Off, 1=Avoid, 2=Boost |
| MANUAL_CONTROL | 15 | int | 0/1 | Manual control mode status |
| ERROR_CODE | 18 | int | 0-121 | Current error code (see [Error Codes](#error-codes)) |
| LOCATING_STATUS | 20 | int | 0-11 | 0=Located, 1=Locating, 10=Failed, 11=Success |
| OBSTACLE_AVOIDANCE | 21 | int | 0/1 | AI obstacle avoidance enabled |
| AI_CAMERA_SETTINGS | 22 | int | varies | AI camera configuration |
| MOP_DOCK_SETTINGS | 23 | int | varies | Mop dock operation mode (serialized) |
| MOP_DOCK_STATUS | 25 | int | 0-6 | Mop dock current state |
| KEY_LOCK | 27 | int | 0/1 | Child lock enabled |
| CARPET_MODE_SENSITIVITY | 28 | int | 0-2 | Carpet detection sensitivity |
| TIGHT_MOP_PATTERN | 29 | int | 0/1 | Enhanced mopping pattern |
| MOP_DOCK_UV_TREATMENT | 32 | int | 0/1 | UV sterilization enabled |
| CARPET_DETECTION_SENSOR | 33 | int | 0/1 | Ultrasonic carpet sensor |
| MOP_DOCK_WET_DRY_SWITCH | 34 | int | varies | Wet/dry vacuum mode |
| CARPET_DETECTION_SENSOR_MODE | 36 | int | 0-2 | Sensor detection mode |
| MOP_DOCK_DETERGENT | 37 | int | 0/1 | Detergent dispenser enabled |
| MOP_DRYING_TIME | 40 | int | hours | Mop drying duration (0-4) |
| MOP_DETACH | 45 | int | 0/1 | Auto mop detach enabled |
| MOP_DOCK_WATER_USAGE | 46 | int | 1-3 | Dock water usage level |
| MISC_TUNABLES | 50 | int | varies | Miscellaneous settings (serialized) |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| START | 1 | Start full cleaning cycle |
| STOP | 2 | Stop current operation |
| MOP_DOCK_INTERACT | 4 | Trigger mop dock operations |

---

### BATTERY (SIID: 3)
Battery status and charging control.

#### Properties
| Property | PIID | Type | Values | Description |
|----------|------|------|--------|-------------|
| LEVEL | 1 | int | 0-100 | Battery percentage |
| CHARGING | 2 | int | 1-5 | 1=Charging, 2=Not charging, 5=Returning |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| START_CHARGE | 1 | Return to dock and charge |

---

### DND (SIID: 5)
Do Not Disturb scheduling.

#### Properties
| Property | PIID | Type | Values | Description |
|----------|------|------|--------|-------------|
| ENABLED | 1 | bool | 0/1 | DND mode enabled |
| START_TIME | 2 | string | "HH:MM" | DND start time (24h format) |
| END_TIME | 3 | string | "HH:MM" | DND end time (24h format) |

---

### MAP (SIID: 6)
Map data and editing operations.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| MAP_DATA | 1 | string | Base64-encoded compressed map |
| FRAME_TYPE | 2 | string | "I" (full) or "P" (partial) |
| CLOUD_FILE_NAME | 3 | string | Cloud storage filename |
| MAP_DETAILS | 4 | string | JSON with room/zone info |
| ACTION_RESULT | 6 | int | Result code of last map edit |

#### Actions
| Action | AIID | Input | Description |
|--------|------|-------|-------------|
| POLL | 1 | frame_type | Request map data |
| EDIT | 2 | edit_params | Edit map (rooms, zones, restrictions) |

**EDIT Action Parameters** (passed as JSON in `in` array):
```json
{
  "piid": 4,
  "value": {
    "req_type": <edit_type>,
    "map_id": <map_id>,
    // Additional parameters based on edit_type
  }
}
```

**Edit Types**:
- `1` = Split segment
- `2` = Merge segments
- `3` = Rename segment
- `4` = Add virtual wall
- `5` = Add no-go zone
- `6` = Add no-mop zone
- `7` = Delete restriction
- `10` = Reset map

---

### AUDIO (SIID: 7)
Audio and voice pack management.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| VOLUME | 1 | int | Volume level (0-100) |
| ACTIVE_VOICEPACK | 2 | string | Current voice pack ID |
| VOICEPACK_INSTALL_STATUS | 3 | int | Installation status |
| INSTALL_VOICEPACK | 4 | string | Voice pack download URL |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| LOCATE | 1 | Play "Find Me" sound |
| VOLUME_TEST | 2 | Test volume level |

---

### MAIN_BRUSH (SIID: 9)
Main brush consumable tracking.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| TIME_LEFT | 1 | int | Hours remaining |
| PERCENT_LEFT | 2 | int | Percentage remaining (0-100) |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### SIDE_BRUSH (SIID: 10)
Side brush consumable tracking.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| TIME_LEFT | 1 | int | Hours remaining |
| PERCENT_LEFT | 2 | int | Percentage remaining (0-100) |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### FILTER (SIID: 11)
Filter consumable tracking.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) ⚠️ Note: Swapped with TIME_LEFT |
| TIME_LEFT | 2 | int | Hours remaining ⚠️ Note: Swapped with PERCENT_LEFT |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### TOTAL_STATISTICS (SIID: 12)
Lifetime cleaning statistics.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| TIME | 2 | int | Total cleaning time (seconds) |
| COUNT | 3 | int | Total cleaning count |
| AREA | 4 | int | Total area cleaned (cm²) |

---

### PERSISTENT_MAPS (SIID: 13)
Multi-floor map management.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| ENABLED | 1 | bool | Persistent maps enabled (0/1) |

---

### AUTO_EMPTY_DOCK (SIID: 15)
Auto-empty dock control (if equipped).

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| AUTO_EMPTY_ENABLED | 1 | bool | Auto-empty enabled (0/1) |
| INTERVAL | 2 | int | Auto-empty interval |
| STATUS | 3 | int | Dock status |
| STATE | 5 | int | 0=Idle, 1=Emptying |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| EMPTY_DUSTBIN | 1 | Manually trigger dust collection |

---

### SENSOR (SIID: 16)
Sensor cleaning reminder.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) |
| TIME_LEFT | 2 | int | Hours remaining |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### SECONDARY_FILTER (SIID: 17)
Secondary filter consumable (if equipped).

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) |
| TIME_LEFT | 2 | int | Hours remaining |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### MOP (SIID: 18)
Mop pad consumable tracking.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) |
| TIME_LEFT | 2 | int | Hours remaining |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### SILVER_ION (SIID: 19)
Silver ion filter consumable (if equipped).

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) |
| TIME_LEFT | 2 | int | Hours remaining |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### DETERGENT (SIID: 20)
Detergent consumable (if equipped).

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| PERCENT_LEFT | 1 | int | Percentage remaining (0-100) |
| TIME_LEFT | 2 | int | Hours remaining |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

### MISC_STATES (SIID: 27)
Dock attachment detection.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| DOCK_FRESHWATER_TANK_ATTACHMENT | 1 | bool | Clean water tank installed (0/1) |
| DOCK_WASTEWATER_TANK_ATTACHMENT | 2 | bool | Wastewater tank installed (0/1) |
| DOCK_DUSTBAG_ATTACHMENT | 3 | bool | Dust bag installed (0/1) |
| DOCK_DETERGENT_ATTACHMENT | 4 | bool | Detergent tank installed (0/1) |

---

### MOP_EXPANSION (SIID: 28)
Advanced mopping features.

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| HIGH_RES_WATER_USAGE | 1 | int | High-resolution water level (1-32) |
| HIGH_RES_MOP_DOCK_HEATER | 8 | int | Dock heater temperature |
| SIDE_BRUSH_ON_CARPET | 29 | int | Side brush on carpet behavior |

---

### WHEEL (SIID: 30)
Wheel module consumable (newer models).

#### Properties
| Property | PIID | Type | Description |
|----------|------|------|-------------|
| TIME_LEFT | 1 | int | Hours remaining |
| PERCENT_LEFT | 2 | int | Percentage remaining (0-100) |

#### Actions
| Action | AIID | Description |
|--------|------|-------------|
| RESET | 1 | Reset consumable timer |

---

## Status and Error Codes

### MODE Values
From `VACUUM_2.MODE` (SIID: 4, PIID: 1):

| Value | Valetudo Status | Flag | Description |
|-------|-----------------|------|-------------|
| 0 | IDLE | - | Idle/Standby |
| 1 | PAUSED | - | Paused during cleaning |
| 2 | CLEANING | - | General cleaning |
| 3 | RETURNING | - | Returning to dock |
| 4 | CLEANING | SEGMENT | Segment cleaning (legacy) |
| 5 | CLEANING | - | Cleaning variant |
| 6 | DOCKED | - | On charging dock |
| 7 | IDLE | - | Idle variant |
| 8 | IDLE | - | Idle variant |
| 9 | IDLE | - | Idle variant |
| 10 | IDLE | - | Idle variant |
| 11 | IDLE | - | Idle variant |
| 12 | IDLE | - | Idle variant |
| 13 | MANUAL_CONTROL | - | Manual/remote control mode |
| 14 | IDLE | - | Power save mode |
| 15 | IDLE | - | Dock self-test/auto-repair |
| 16 | IDLE | - | Idle variant |
| 17 | IDLE | - | Idle variant |
| 18 | CLEANING | SEGMENT | Room/segment cleaning |
| 19 | CLEANING | ZONE | Zone cleaning |
| 20 | CLEANING | SPOT | Spot cleaning |
| 21 | MOVING | MAPPING | Mapping run |
| 23 | MOVING | TARGET | Going to target location |

---

### Error Codes
From `VACUUM_2.ERROR_CODE` (SIID: 4, PIID: 18):

| Code | Severity | Subsystem | Description |
|------|----------|-----------|-------------|
| 0 | - | - | No error |
| 1 | Error | Core | Wheel lost floor contact |
| 2 | Error | Sensors | Cliff sensor dirty or robot on edge |
| 3 | Error | Sensors | Stuck front bumper |
| 4 | Warning | Sensors | Tilted robot |
| 5 | Error | Sensors | Stuck front bumper |
| 6 | Error | Core | Wheel lost floor contact |
| 7 | Unknown | Core | Internal error |
| 8 | Warning | Attachments | Dustbin missing |
| 9 | Warning | Attachments | Water tank missing |
| 10 | Warning | Attachments | Water tank empty |
| 11 | Warning | Attachments | Dustbin full |
| 12 | Error | Motors | Main brush jammed |
| 13 | Error | Motors | Side brush jammed |
| 14 | Warning | Attachments | Filter jammed |
| 15 | Warning | Navigation | Robot stuck or trapped |
| 16 | Warning | Navigation | Robot stuck or trapped |
| 17 | Warning | Navigation | Robot stuck or trapped |
| 18 | Warning | Navigation | Robot stuck or trapped |
| 19 | Warning | Power | Charging station without power |
| 20 | Info | Power | Low battery |
| 21 | Warning | Power | Charging error |
| 23 | Unknown | Core | Internal error (heart status) |
| 24 | Warning | Sensors | Camera dirty |
| 26 | Warning | Sensors | Camera dirty |
| 27 | Warning | Sensors | Sensor dirty |
| 28 | Warning | Power | Charging station without power |
| 29 | Error | Power | Battery temperature out of range |
| 30 | Critical | Motors | Fan speed abnormal |
| 31 | Warning | Navigation | Robot stuck or trapped |
| 32 | Warning | Navigation | Robot stuck or trapped |
| 33 | Critical | Sensors | Accelerometer sensor error |
| 34 | Critical | Sensors | Gyroscope sensor error |
| 35 | Critical | Sensors | Gyroscope sensor error |
| 36 | Critical | Sensors | Left magnetic field sensor error |
| 37 | Critical | Sensors | Right magnetic field sensor error |
| 40 | Critical | Sensors | Camera fault |
| 41 | Info | Sensors | Magnetic interference |
| 42 | Critical | Attachments | Water pump fault |
| 43 | Critical | Core | RTC fault |
| 45 | Critical | Core | 3.3V rail abnormal |
| 47 | Warning | Navigation | Cannot reach target |
| 48 | Error | Sensors | LDS jammed |
| 49 | Error | Sensors | LDS bumper jammed |
| 51 | Warning | Attachments | Filter jammed |
| 53 | Critical | Sensors | ToF sensor offline |
| 54 | Warning | Sensors | Wall sensor dirty |
| 59 | Warning | Navigation | Robot trapped by virtual restrictions |
| 61 | Warning | Navigation | Cannot reach target |
| 62 | Warning | Navigation | Cannot reach target |
| 63 | Warning | Navigation | Cannot reach target |
| 64 | Warning | Navigation | Cannot reach target |
| 65 | Warning | Navigation | Cannot reach target |
| 66 | Warning | Navigation | Cannot reach target |
| 67 | Warning | Navigation | Cannot reach target |
| 68 | Info | - | Docked with mop still attached (reminder) |
| 69 | Error | Attachments | Lost mop pad |
| 70 | Error | Attachments | Lost mop pad |
| 71 | Critical | Motors | Mop motor fault |
| 72 | Critical | Motors | Mop motor current abnormal |
| 74 | Error | Attachments | Failed to attach mop pads |
| 91 | Warning | Navigation | Cannot reach target |
| 96 | Warning | Navigation | Cannot reach target |
| -2 | Warning | Navigation | Stuck inside restricted area |
| 101 | Warning | Dock | Auto-empty dust bag full or duct clogged |
| 102 | Warning | Dock | Auto-empty cover open or missing dust bag |
| 103 | Warning | Dock | Auto-empty cover open or missing dust bag |
| 104 | Warning | Dock | Auto-empty dust bag full or duct clogged |
| 105 | Warning | Dock | Mop dock clean water tank not installed |
| 106 | Warning | Dock | Mop dock wastewater tank not installed/full |
| 107 | Warning | Dock | Mop dock clean water tank empty |
| 108 | Warning | Dock | Mop dock wastewater tank not installed/full |
| 109 | Error | Dock | Mop dock wastewater pipe clogged |
| 110 | Critical | Dock | Mop dock wastewater pump damaged |
| 111 | Warning | Dock | Mop dock tray not installed |
| 112 | Error | Dock | Mop dock tray full of water |
| 114 | Info | - | Reminder to clean mop tray |
| 116 | Warning | Dock | Mop dock clean water tank empty |
| 117 | Warning | Navigation | Cannot navigate to dock |
| 118 | Warning | Dock | Mop dock wastewater tank not installed/full |
| 119 | Error | Dock | Mop dock tray full of water |
| 121 | Warning | Dock | Auto-empty dust bag full or duct clogged |
| 122 | Info | - | Water hookup test succeeded |

---

### Mop Dock Status Values
From `VACUUM_2.MOP_DOCK_STATUS` (SIID: 4, PIID: 25):

| Value | Status | Description |
|-------|--------|-------------|
| 0 | IDLE | Dock idle |
| 1 | CLEANING | Washing mop pads |
| 2 | DRYING | Drying mop pads |
| 3 | CLEANING | Cleaning variant |
| 4 | PAUSE | Paused |
| 5 | CLEANING | Cleaning variant |
| 6 | CLEANING | Cleaning variant |

---

### Auto-Empty Dock Status Values
From `AUTO_EMPTY_DOCK.STATE` (SIID: 15, PIID: 5):

| Value | Status | Description |
|-------|--------|-------------|
| 0 | IDLE | Not emptying |
| 1 | EMPTYING | Dust collection in progress |
| 2 | IDLE | DND mode active |

---

## Map Data Flow

### 1. Map Upload Process

The robot pushes map data to Valetudo's FDS Mock Server:

```
Robot Firmware
      │
      │ HTTP PUT
      ▼
http://127.0.0.1:8079/api/miio/fds_upload_handler
      │
      ▼
DreameMapParser
      │
      ├─► Base64 decode
      ├─► Character replacement (swap special chars)
      ├─► zlib decompression
      └─► Binary parsing
      │
      ▼
ValetudoMap object
```

### 2. Map Request (Pull)

Valetudo can request map data using the MAP.POLL action:

```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 6,
    "aiid": 1,
    "in": [{
      "piid": 2,
      "value": "{\"frame_type\":\"I\", \"force_type\": 1, \"req_type\": 1}"
    }]
  }
}
```

**Parameters**:
- `frame_type`: `"I"` for full map (I-Frame), `"P"` for partial update (P-Frame)
- `force_type`: `1` to force a new frame
- `req_type`: `1` for standard request

**Response** (simplified):
```json
{
  "code": 0,
  "out": [{
    "piid": 1,
    "value": "<base64_zlib_compressed_binary_map_data>"
  }]
}
```

### 3. Map Data Format

The binary map data contains:
- **Header**: Map version, dimensions, resolution
- **Layers**:
  - Floor pixels
  - Wall pixels
  - Segments (rooms)
  - Virtual walls
  - No-go zones
  - No-mop zones
  - Cleaning paths
  - Robot position
  - Charger position
  - Obstacles (if AI-enabled)

### 4. Map Editing

Map edits are sent via `MAP.EDIT` action:

```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 6,
    "aiid": 2,
    "in": [{
      "piid": 4,
      "value": {
        "req_type": 3,
        "map_id": 1234567890,
        "segment_id": 18,
        "name": "Kitchen"
      }
    }]
  }
}
```

**Common Edit Operations**:
- Rename room: `req_type: 3`
- Add virtual wall: `req_type: 4`
- Add no-go zone: `req_type: 5`
- Add no-mop zone: `req_type: 6`
- Delete restriction: `req_type: 7`
- Reset map: `req_type: 10`

---

## Advanced Topics

### Zone Cleaning

To start zone cleaning, use `VACUUM_2.START` with `ADDITIONAL_CLEANUP_PROPERTIES`:

```json
{
  "method": "set_properties",
  "params": [{
    "did": 277962183,
    "siid": 4,
    "piid": 1,
    "value": 19
  }, {
    "did": 277962183,
    "siid": 4,
    "piid": 10,
    "value": "{\"areas\":[{\"x1\":25000,\"y1\":25000,\"x2\":30000,\"y2\":30000,\"count\":1}]}"
  }]
}
```

Then trigger:
```json
{
  "method": "action",
  "params": {
    "did": 277962183,
    "siid": 4,
    "aiid": 1
  }
}
```

### Segment Cleaning

To clean specific rooms:

```json
{
  "method": "set_properties",
  "params": [{
    "did": 277962183,
    "siid": 4,
    "piid": 1,
    "value": 18
  }, {
    "did": 277962183,
    "siid": 4,
    "piid": 10,
    "value": "{\"segments\":[{\"segment_id\":16,\"repeat\":1},{\"segment_id\":17,\"repeat\":2}]}"
  }]
}
```

Then trigger the start action.

---

## Implementation Notes

### Token and Device ID

- **Device ID**: Found in `/data/config/miio/device.conf` (field: `did`)
- **Local Token**: 16-byte token from `/data/config/miio/device.token`
- **Cloud Secret**: 16-byte key from `/data/config/miio/device.conf` (field: `key`)

### Encryption

All UDP communication is encrypted using AES-128-CBC with the device token/cloud secret as the key.

### Polling Intervals

Valetudo polls robot state:
- **Active state** (cleaning/moving): Every 3 seconds
- **Idle/Docked**: Every 30 seconds
- **Map data**: On-demand when state changes

### Push Notifications

The robot sends `properties_changed` messages via the cloud socket when values change:

```json
{
  "method": "properties_changed",
  "params": [{
    "siid": 3,
    "piid": 1,
    "value": 84
  }]
}
```

Valetudo acknowledges with:
```json
{
  "id": <msg_id>,
  "result": "ok"
}
```

---

## References

- **Official MIoT Spec**: [https://miot-spec.org](https://miot-spec.org)
- **Your Robot Model**: `urn:miot-spec-v2:device:vacuum:0000A006:dreame-p2029:1`
- **Valetudo Documentation**: [https://valetudo.cloud](https://valetudo.cloud)
- **Source Code**:
  - `backend/lib/robots/dreame/DreameL10ProValetudoRobot.js`
  - `backend/lib/robots/dreame/DreameMiotServices.js`
  - `backend/lib/robots/dreame/DreameMiotHelper.js`
  - `backend/lib/robots/MiioValetudoRobot.js`

---

**Document Version**: 1.0
**Generated**: 2025-10-15
**For**: Dreame L10 Pro (dreame.vacuum.p2029)
**MIoT Generation**: GEN2
