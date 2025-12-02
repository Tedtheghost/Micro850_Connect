# Heartbeat Monitoring Setup Guide

## Overview
The Connection monitoring program now uses a **heartbeat monitoring method** to reliably detect Ethernet/IP connection health. This method monitors an incrementing counter value from a remote device to verify active communication.

## What Changed

### Fixed Issues
- **Compiler Error Fixed**: Removed invalid `.0` bit accessor from `_IO_EM_DO_01` (BOOL variables cannot use bit addressing)
- **Enhanced Monitoring**: Implemented robust heartbeat-based connection detection

### New Variables Added
```
remoteHeartbeat_In : UDINT    // Input from remote device
lastHeartbeat : UDINT         // Previous heartbeat value
heartbeatChanged : BOOL       // Indicates heartbeat value changed
heartbeatTimeout : TON        // 5-second timeout timer
heartbeatOK : BOOL           // Heartbeat is active and valid
```

## How It Works

1. **Heartbeat Counter**: A remote device (HMI, SCADA, or another PLC) continuously increments a counter value
2. **Change Detection**: The PLC monitors if this counter value changes each scan
3. **Timeout Protection**: If the counter doesn't change for 5 seconds, connection is considered lost
4. **Combined Status**: Connection is valid when BOTH heartbeat is active AND NetworkStatus_In is TRUE

## Setup Instructions

### Step 1: Configure Remote Device (HMI/SCADA/PLC)

You need to set up a counter on your remote device that increments continuously:

#### For FactoryTalk View / PanelView HMI:
```
// Create a tag that increments every second
HeartbeatCounter (DINT)

// In a macro or periodic task:
HeartbeatCounter = HeartbeatCounter + 1
IF HeartbeatCounter > 65000 THEN
    HeartbeatCounter = 0
END IF
```

#### For SCADA/OPC Client:
- Create a tag that writes to the PLC's `remoteHeartbeat_In` variable
- Increment this value periodically (every 500ms - 2s recommended)
- Allow rollover at a reasonable value (e.g., 65535)

#### For Another PLC:
```
// In your remote PLC program
VAR
    heartbeatCounter : UDINT := 0;
    heartbeatTimer : TON;
END_VAR

// Increment every second
heartbeatTimer(IN := NOT heartbeatTimer.Q, PT := T#1s);
IF heartbeatTimer.Q THEN
    heartbeatCounter := heartbeatCounter + 1;
END_IF

// Map this to the Micro 850's input via EtherNet/IP
```

### Step 2: Map the Heartbeat Input in CCW

1. **Open your CCW project**
2. **Configure EtherNet/IP Connection**:
   - Add your remote device to the I/O Configuration
   - Create a connection to the device

3. **Map the Heartbeat Variable**:
   - In the I/O mapping, assign the remote heartbeat counter to the `remoteHeartbeat_In` variable
   - Example: Map remote device's heartbeat tag → `Connection.remoteHeartbeat_In`

### Step 3: Verify NetworkStatus_In

Ensure `NetworkStatus_In` is connected to a valid network status indicator:
- Could be a module health bit
- Could be a manual input for testing
- Could be tied to another diagnostic signal

### Step 4: Testing

1. **Download the updated program** to your Micro 850
2. **Monitor these variables** in CCW online mode:
   - `remoteHeartbeat_In` - Should increment continuously
   - `heartbeatChanged` - Should toggle TRUE/FALSE each scan
   - `heartbeatOK` - Should be TRUE when connection is healthy
   - `ethernetLinkStatus` - Final connection status

3. **Test failure scenarios**:
   - Disconnect network cable → `ethernetLinkStatus` should go FALSE within 5 seconds
   - Stop remote heartbeat → Connection should be detected as lost
   - Restore connection → Should reconnect automatically

## Adjusting Timeout Period

The default timeout is **5 seconds**. To adjust:

In `Connection.stf`, line ~35:
```
heartbeatTimeout(IN := NOT heartbeatChanged, PT := T#5s);
```

Change `T#5s` to your desired timeout:
- `T#3s` for faster detection (3 seconds)
- `T#10s` for slower, more stable detection (10 seconds)
- Recommended range: 3-10 seconds

## Monitoring and Diagnostics

### Real-Time Monitoring
Watch these variables in CCW to monitor connection health:
- `ethernetLinkStatus` - Overall connection status
- `heartbeatOK` - Heartbeat communication active
- `networkStatusOK` - Network module status
- `remoteHeartbeat_In` - Current heartbeat value (should increment)

### Report Generation
Generate a report by setting `GenerateReport := TRUE`. The report includes:
- Connection status (CONNECTED / NO PHYSICAL LINK / NETWORK ERROR)
- Link status (ACTIVE / DOWN)
- Module status (ONLINE / FAULT)
- **Heartbeat status (ACTIVE / TIMEOUT)** ← New!
- Statistics and diagnostics

View the report parts:
- `reportPart1` - Status overview
- `reportPart2` - Statistics
- `reportPart3` - Timing and diagnostics

## Troubleshooting

### Heartbeat Always Shows TIMEOUT
**Problem**: `heartbeatOK` is always FALSE
**Solutions**:
- Verify remote device is incrementing the counter
- Check EtherNet/IP connection is established
- Verify `remoteHeartbeat_In` is mapped correctly in I/O configuration
- Confirm remote counter is updating faster than 5 seconds

### Connection Flickers On/Off
**Problem**: `ethernetLinkStatus` toggles rapidly
**Solutions**:
- Increase timeout period (T#10s instead of T#5s)
- Check network stability (switches, cables)
- Verify remote heartbeat updates consistently
- Check for PLC scan time issues

### Heartbeat Value Doesn't Change
**Problem**: `remoteHeartbeat_In` stays constant
**Solutions**:
- Verify remote device program is running
- Check I/O mapping in CCW project
- Confirm EtherNet/IP connection is configured
- Test with a manual value change in online mode

## Benefits of Heartbeat Method

✅ **Detects actual communication**, not just physical link
✅ **Catches network issues** beyond cable disconnection
✅ **No special hardware required** - works with any EtherNet/IP device
✅ **Configurable timeout** for your application needs
✅ **Robust fault detection** - identifies stalled networks
✅ **Works with implicit/explicit messaging**

## Questions?

If you need help:
1. Check variable values in CCW online mode
2. Verify remote device heartbeat is incrementing
3. Review I/O configuration mappings
4. Test with a simulated heartbeat value
