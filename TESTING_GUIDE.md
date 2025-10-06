# Testing Guide for MSP_SET_RAW_MOTORS

## Prerequisites
1. Betaflight firmware compiled with `USE_RX_MSP_OVERRIDE` feature enabled
2. Flight controller with the new firmware flashed
3. MSP tool or custom script capable of sending MSP commands
4. Test bench setup (motors disconnected from props for safety!)

## Setup Steps

### 1. Configure MSP_OVERRIDE Mode
Using Betaflight Configurator:
1. Go to the "Modes" tab
2. Find "MSP OVERRIDE" mode
3. Assign it to an AUX channel (e.g., AUX1)
4. Set the range (e.g., 1700-2100 for the high position of a switch)
5. Save configuration

### 2. Configure MSP Override Channels (Optional)
In the CLI:
```
set msp_override_channels_mask = 15  # Override channels 1-4 (AETR)
save
```

## Testing Procedure

### Test 1: Basic Functionality
1. **Safety**: Remove propellers!
2. Arm the flight controller
3. Activate MSP_OVERRIDE mode via the configured AUX channel
4. Send MSP_SET_RAW_MOTORS command with test values
5. Observe motor response
6. Deactivate MSP_OVERRIDE mode
7. Verify motors return to normal mixer control

### Test 2: Safety Checks

#### Test 2a: Unarmed Protection
1. Ensure flight controller is DISARMED
2. Send MSP_SET_RAW_MOTORS command
3. **Expected**: Command is ignored, motors do not respond

#### Test 2b: Mode Inactive Protection
1. Arm the flight controller
2. Ensure MSP_OVERRIDE mode is INACTIVE
3. Send MSP_SET_RAW_MOTORS command
4. **Expected**: Command is ignored, normal mixer control continues

#### Test 2c: Automatic Reset on Disarm
1. Arm and activate MSP_OVERRIDE mode
2. Send MSP_SET_RAW_MOTORS command with specific values
3. Disarm the flight controller
4. Re-arm and activate MSP_OVERRIDE mode
5. **Expected**: Previous values cleared, motors at normal idle

#### Test 2d: Automatic Reset on Mode Deactivation
1. Arm and activate MSP_OVERRIDE mode
2. Send MSP_SET_RAW_MOTORS command
3. Deactivate MSP_OVERRIDE mode (while still armed)
4. **Expected**: Motors immediately return to normal mixer control

### Test 3: Motor Value Ranges
Test with different motor value ranges:
- Minimum: 1000 µs (stopped)
- Low throttle: 1100 µs
- Mid throttle: 1500 µs
- High throttle: 1900 µs
- Maximum: 2000 µs

## MSP Command Format

### MSP_SET_RAW_MOTORS (213)
**Payload structure** (for 4 motors):
```
[M1_low, M1_high, M2_low, M2_high, M3_low, M3_high, M4_low, M4_high]
```

Each motor value is a uint16 in little-endian format, representing PWM microseconds.

### Example: Python with pymsp
```python
import serial
import struct

# MSP command IDs
MSP_SET_RAW_MOTORS = 213

def send_raw_motors(port, motor_values):
    """
    Send raw motor values.
    motor_values: list of 4 integers (PWM microseconds, typically 1000-2000)
    """
    # Pack motor values as uint16 little-endian
    payload = struct.pack('<4H', *motor_values)
    
    # Calculate MSP packet (simplified)
    size = len(payload)
    checksum = size ^ MSP_SET_RAW_MOTORS
    for byte in payload:
        checksum ^= byte
    
    # MSP v1 packet format
    packet = b'$M<'
    packet += bytes([size, MSP_SET_RAW_MOTORS])
    packet += payload
    packet += bytes([checksum])
    
    port.write(packet)

# Example usage
with serial.Serial('/dev/ttyUSB0', 115200, timeout=1) as ser:
    # Set motors to low idle (1100 µs)
    send_raw_motors(ser, [1100, 1100, 1100, 1100])
    
    # Set motors to different values
    send_raw_motors(ser, [1200, 1300, 1400, 1500])
    
    # Stop all motors (disarm or deactivate mode first!)
    send_raw_motors(ser, [1000, 1000, 1000, 1000])
```

### Example: betaflight-configurator JavaScript
```javascript
// Assuming MSP is available
const MSP_SET_RAW_MOTORS = 213;

function sendRawMotors(motorValues) {
    // motorValues: array of 4 integers [m1, m2, m3, m4]
    const buffer = [];
    motorValues.forEach(value => {
        buffer.push(value & 0xFF);          // low byte
        buffer.push((value >> 8) & 0xFF);   // high byte
    });
    
    MSP.send_message(MSP_SET_RAW_MOTORS, buffer, false);
}

// Example: Set all motors to 1500 µs
sendRawMotors([1500, 1500, 1500, 1500]);
```

## Expected Behavior

### Normal Operation
- Motors respond immediately to commanded values
- No PID corrections applied
- No mixer calculations applied
- Direct PWM output to ESCs

### With Props (EXTREME CAUTION!)
⚠️ **WARNING**: Only test with props in a controlled environment with proper safety equipment!
- Aircraft will not stabilize itself
- Manual control required
- Risk of immediate crash if values not balanced

## Troubleshooting

### Issue: Motors don't respond
**Check:**
1. Is the FC armed?
2. Is MSP_OVERRIDE mode active? (check in Configurator OSD)
3. Is USE_RX_MSP_OVERRIDE compiled into firmware?
4. Are motor values in valid range (1000-2000)?

### Issue: Motors respond but then stop
**Likely cause:** Mode was deactivated or FC was disarmed
**Solution:** Re-arm and re-activate mode

### Issue: Command accepted but motors still follow normal control
**Likely cause:** Raw motor override not active
**Check:** Ensure both armed AND MSP_OVERRIDE mode is active before sending command

## Safety Reminders
1. ⚠️ ALWAYS remove propellers for initial testing
2. ⚠️ Use a proper test bench with motor restraints
3. ⚠️ Never test with props near people or property
4. ⚠️ This bypasses all safety features - use with extreme caution
5. ⚠️ Ensure proper cooling if running motors at high throttle
6. ⚠️ Monitor motor/ESC temperatures during extended testing

## Advanced Testing

### Test Motor Response Curves
Send incremental values and measure motor speed:
```python
for throttle in range(1000, 2000, 100):
    send_raw_motors(ser, [throttle] * 4)
    time.sleep(2)  # Allow time for measurement
```

### Test Individual Motor Control
```python
# Test each motor individually
for motor_idx in range(4):
    values = [1000] * 4
    values[motor_idx] = 1500
    send_raw_motors(ser, values)
    time.sleep(1)
```

### Test Motor Synchronization
Send synchronized commands and verify timing:
```python
import time

# Rapidly change all motors together
for _ in range(10):
    send_raw_motors(ser, [1200, 1200, 1200, 1200])
    time.sleep(0.1)
    send_raw_motors(ser, [1400, 1400, 1400, 1400])
    time.sleep(0.1)
```

## Data Logging
Consider enabling Blackbox logging to verify:
- Motor output values match commanded values
- Mode activation/deactivation timing
- Transition between normal and override control

## Success Criteria
✅ Motors respond only when armed AND MSP_OVERRIDE active
✅ Values reset automatically on disarm
✅ Values reset automatically on mode deactivation
✅ No response when safety conditions not met
✅ Smooth transition between normal and override control
