# SMC JXCL Electric Actuator Controller

Made with love by the **CristiMediaGroup** team!
Fixed and enhanced by Claude.

## Overview

This Siemens TIA Portal project controls an **SMC JXCL18-LEL25MT-300** electric actuator via IO-Link. The actuator performs an automatic cycle:

1. Homing - establishes origin position
2. Forward movement - moves to target position (200.00 mm)
3. 3-second delay - waits at target position
4. Return to home - moves back to origin (0 mm)

## Hardware

| Component | Model |
|-----------|-------|
| PLC | Siemens S7-1200 (CPU 1215C DC/DC/Rly) |
| Actuator | SMC JXCL18-LEL25MT-300 (300 mm stroke) |
| Communication | IO-Link |

## Project Structure

| Block | Type | Description |
|-------|------|-------------|
| `JXCL` | FB3 | Main state machine (this code) |
| `JXCL_DB` | DB4 | Instance DB for JXCL FB |
| `Uitgangen` | FC2 | Output coordinator |
| `Status` | FC1 | Status monitoring |
| `S` | DB3 | Sensor data block |
| `HM` | DB5 | HMI data block |

## FB Interface: `JXCL`

### Inputs

| Name | Type | Source | Description |
|------|------|--------|-------------|
| `Motor_Plus` | Bool | `%DB3.DBX1.3` `"S".S7` | Start command (HMI/Method 1) |
| `Motor_Plus_2` | Bool | - | Start command (Method 2) |
| `Reset_Cmd` | Bool | `%DB5.DBX0.6` `"HM".Reset JXCLHM` | Alarm reset |

### Outputs

| Name | Type | Destination | Description |
|------|------|-------------|-------------|
| `Done` | Bool | `#Actuator_Home_Done` | Cycle complete |
| `Forward_Done` | Bool | `%M0.0` `"JXCL+"` | Forward position reached |
| `Home_Done` | Bool | `%M0.1` `"JXCL-"` | Home position reached |

### Static Variables

| Name | Type | Description |
|------|------|-------------|
| `State` | Int | State machine current state |
| `Step` | Int | Reserved |
| `Delay_Counter` | Int | Scan counter for 3s delay (~100ms PLC cycle) |

## State Machine

| State | Name | Description |
|-------|------|-------------|
| 0 | IDLE | Wait for start command, reset outputs |
| 1 | SERVO ON | Enable servo, wait for SVRE |
| 2 | HOMING | Execute SETUP until SETON = TRUE |
| 10 | PARAMS FORWARD | Write movement parameters (Target = 20000) |
| 11 | LATCH | One scan delay for stable parameters |
| 12 | START FORWARD | Set RUN = 1, trigger movement |
| 13 | RUN OFF (BUSY) | Clear RUN when BUSY rises |
| 14 | WAIT INP | Wait for position reached (INP = TRUE) |
| 15 | 3S DELAY | Count 30 scans (~3 seconds) |
| 20 | PARAMS HOME | Write home parameters (Target = 0) |
| 21 | LATCH | One scan delay |
| 22 | START HOME | Set RUN = 1, trigger return |
| 23 | RUN OFF (BUSY) | Clear RUN when BUSY rises |
| 24 | WAIT HOME | Wait for home position reached |
| 30 | DONE | Set Done = TRUE, wait for buttons released |
| 99 | ERROR | Alarm handling with manual reset |

## Key Features

- Dual start methods: Two independent start inputs (`Motor_Plus` OR `Motor_Plus_2`)
- Automatic cycle: Forward -&gt; Delay -&gt; Home without intervention
- Safe RUN pulse: RUN bit cleared immediately after BUSY rises (per SMC manual)
- Alarm reset: Manual reset via `Reset_Cmd` input
- ESTOP/ALARM protection: Cycle blocked if safety signals active

## Process Data Mapping

### Outputs (`PD_OUT_data`)

| Signal | Description |
|--------|-------------|
| `SVON` | Servo ON |
| `SETUP` | Homing trigger |
| `RUN` | Movement start (pulse) |
| `RESET` | Alarm reset |
| `Move` | Movement mode (1 = ABS) |
| `Speed` | Target speed (100 = 100 mm/s) |
| `Target Pos` | Target position (0.01 mm units) |
| `Accel` | Acceleration |
| `Decel` | Deceleration |
| `PushingF` | Pushing force (0 = positioning) |
| `TriggerLv` | Trigger level |
| `InPosition` | In-position window (0.01 mm) |

### Inputs (`PD_IN_data`)

| Signal | Description |
|--------|-------------|
| `SVRE` | Servo ready |
| `SETON` | Origin established |
| `BUSY` | Movement in progress |
| `INP` | In position |
| `ALARM` | Alarm active |
| `ESTOP` | Emergency stop |

## Position Values

| Position | Value | Note |
|----------|-------|------|
| Home | 0 | Origin after homing |
| Forward | 20000 | 200.00 mm (0.01 mm units) |
| Full stroke | 30000 | 300.00 mm (actuator maximum) |

## Timing

| Parameter | Value |
|-----------|-------|
| PLC cycle time | ~100 ms |
| Delay counter | 30 scans |
| Effective delay | ~3 seconds |

## Installation

1. Import FB `JXCL` into your TIA Portal project
2. Create instance DB `JXCL_DB`
3. Map IO-Link process data tags
4. Connect start inputs (`Motor_Plus`, `Motor_Plus_2`)
5. Download to PLC and test

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| No movement | `Move = 0` invalid | Check `Move = 1` (ABS) |
| Alarm on start | Target out of range | Verify `Target Pos` &lt;= 30000 |
| RUN ignored | Parameters changing while RUN high | Ensure RUN cleared when BUSY rises |
| Stuck in State 3 | Waiting for SETON low | SETON stays HIGH - remove State 3 |
| Timer errors | `IEC_TIMER` incompatible | Use scan counter method |

Made with ❤️ by CristiMediaGroup
