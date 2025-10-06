# MSP_SET_RAW_MOTORS Command Usage

## Overview
This document describes the new `MSP_SET_RAW_MOTORS` (command ID: 213) command that allows sending raw motor values that bypass the mixer when the MSP_OVERRIDE mode is active.

## Command Details

### Command ID
- **MSP_SET_RAW_MOTORS**: 213

### Direction
- **In message** (from ground station/configurator to flight controller)

### Prerequisites
To use this command, the following conditions must be met:
1. The flight controller must be **ARMED**
2. The **MSP_OVERRIDE** mode must be **ACTIVE** (BOXMSPOVERRIDE enabled)
3. The firmware must be compiled with `USE_RX_MSP_OVERRIDE` feature enabled

### Payload Format
The command expects motor values for all motors in sequence:
- Each motor value is sent as a **uint16** (2 bytes, little-endian)
- Values are in the same external format as used by MSP_MOTOR and MSP_SET_MOTOR commands
- The number of motor values must match the motor count (typically 4 for quadcopters)

Example for a quadcopter (4 motors):
```
[motor1_low, motor1_high, motor2_low, motor2_high, motor3_low, motor3_high, motor4_low, motor4_high]
```

### Behavior

#### When Prerequisites are Met (Armed + MSP_OVERRIDE active)
1. The command stores the received motor values in `rawMotorOverride[]` array
2. Sets `rawMotorOverrideActive = true`
3. During the next mixer cycle (`mixTable()`), the raw motor values bypass the mixer and are directly applied to the motors
4. The normal PID controller and mixer calculations are skipped for motor output

#### When Prerequisites are NOT Met (Not Armed OR MSP_OVERRIDE inactive)
1. The command data is consumed but NOT applied
2. `rawMotorOverrideActive` is set to `false`
3. Normal mixer operation continues

### Safety Features

#### Automatic Reset on Disarm
When the flight controller is disarmed, the raw motor override is automatically reset by calling `mixerResetRawMotorOverride()`.

#### Automatic Reset on Mode Change
If MSP_OVERRIDE mode is deactivated while armed, the raw motor override is automatically reset at the start of the next mixer cycle.

#### Mode Check in Mixer
Even if raw motor override is active, it will only be applied if:
- `rawMotorOverrideActive == true`
- `IS_RC_MODE_ACTIVE(BOXMSPOVERRIDE) == true`
- `ARMING_FLAG(ARMED) == true`

If any of these conditions become false, the mixer reverts to normal operation.

## Use Cases

This command is useful for:
1. **Motor testing**: Direct control of individual motors for testing and calibration
2. **Custom control algorithms**: Implementing custom flight controllers that bypass Betaflight's mixer
3. **Research and development**: Testing motor responses and developing new control strategies
4. **Motor synchronization testing**: Verifying motor timing and responses

## Example Workflow

1. Enable MSP_OVERRIDE mode on an AUX channel in Betaflight Configurator
2. Arm the flight controller
3. Activate MSP_OVERRIDE mode via the configured AUX channel
4. Send MSP_SET_RAW_MOTORS command with desired motor values
5. Motors will respond directly to the commanded values, bypassing the mixer
6. To return to normal flight, deactivate MSP_OVERRIDE mode
7. When disarmed, the raw motor override is automatically cleared

## Implementation Details

### Files Modified
- `src/main/msp/msp_protocol.h` - Added MSP_SET_RAW_MOTORS command definition
- `src/main/flight/mixer.h` - Added external declarations for rawMotorOverride arrays and reset function
- `src/main/flight/mixer.c` - Implemented raw motor override logic and reset function
- `src/main/msp/msp.c` - Added MSP_SET_RAW_MOTORS command handler
- `src/main/fc/core.c` - Added reset call on disarm

### Global Variables
- `float rawMotorOverride[MAX_SUPPORTED_MOTORS]` - Stores raw motor values
- `bool rawMotorOverrideActive` - Flag indicating if raw override is active

### Functions
- `void mixerResetRawMotorOverride(void)` - Resets raw motor override to inactive state

## Notes

- The motor values are converted from external format (PWM range) to internal format (0.0 to 1.0) using `motorConvertFromExternal()`
- The raw motor values are applied in the same coordinate system as the normal mixer output
- This feature requires the `USE_RX_MSP_OVERRIDE` build flag to be enabled
