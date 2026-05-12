# Loopback-Only Bonjour Implementation Summary

## Overview

This implementation restricts iMCP's Bonjour service advertisement to loopback-only by default, preventing cross-machine discovery and addressing the silent multi-instance problem described in the issue. Users can opt-in to LAN access via a new preference toggle.

## Changes Made

### 1. ServerController.swift

#### Added Network Preference Storage
- **Line 180**: Added `@AppStorage("allowLANConnections") private var allowLANConnections = false`
  - Defaults to `false` for loopback-only operation
  - Persists across app launches

#### Added LAN Connection Management Method
- **Lines 243-246**: Added `setAllowLANConnections(_ allowed: Bool)` method
  - Bridges UI changes to the network manager
  - Called when the user toggles the preference

#### Updated Initialization
- **Line 276**: Added call to `await networkManager.setAllowLANConnections(self.allowLANConnections)`
  - Ensures the network manager starts with the correct configuration
  - Applied before the server starts listening

### 2. NetworkDiscoveryManager Actor

#### Added State Management
- **Line 551**: Added `private var allowLANConnections: Bool = false`
  - Tracks the current LAN connection setting

#### Refactored Parameter Creation
- **Lines 571-593**: Created `createParameters(allowLAN: Bool)` static method
  - Returns `NWParameters` configured for either loopback-only or LAN access
  - When `allowLAN = false`: Sets `acceptLocalOnly = true` (restricts to 127.0.0.1)
  - When `allowLAN = true`: Sets `acceptLocalOnly = false` (allows all network interfaces)
  - Both modes disable peer-to-peer and restrict to IPv4

#### Updated Initialization
- **Line 559**: Changed to use `createParameters(allowLAN: false)` by default
  - Ensures service starts in loopback-only mode

#### Updated Listener Restart Logic
- **Line 619**: Modified `restartWithRandomPort()` to use `createParameters(allowLAN: allowLANConnections)`
  - Preserves the current LAN setting when restarting after port conflicts
  - Log message now includes the LAN setting state

#### Added Dynamic Configuration Method
- **Lines 640-651**: Added `setAllowLANConnections(_ allowed: Bool)` method
  - Checks if the setting has actually changed (avoids unnecessary restarts)
  - Updates the `allowLANConnections` state
  - Restarts the listener with new parameters via `restartWithRandomPort()`

### 3. ServerNetworkManager Actor

#### Added Configuration Method
- **Lines 1062-1074**: Added `setAllowLANConnections(_ allowed: Bool)` method
  - Validates that the discovery manager is initialized
  - Delegates to `NetworkDiscoveryManager.setAllowLANConnections()`
  - Handles errors gracefully with logging

### 4. SettingsView.swift

#### Added Network Settings Section
- **Line 72**: Added `@AppStorage("allowLANConnections")` to `GeneralSettingsView`
  - Binds directly to the same UserDefaults key as ServerController

#### Added Network Access UI
- **Lines 80-119**: Added "Network Access" section with toggle
  - Clear header and description
  - Toggle with custom binding that calls `serverController.setAllowLANConnections()`
  - Contextual help text that changes based on the current setting
  - Placed above "Trusted Clients" section for logical flow

## Security Benefits

1. **Default Security**: New installations start in loopback-only mode
2. **Explicit Opt-In**: Users must deliberately enable LAN access
3. **Clear Communication**: UI explains security implications
4. **Partial Resolution of #103**: Addresses the security concern by default

## How It Works

### Loopback-Only Mode (Default)
1. `NWListener` is created with `acceptLocalOnly = true`
2. Bonjour service advertises on `_mcp._tcp.local.`
3. Service is **only visible** on the local machine (127.0.0.1)
4. `imcp-server` CLI on the same Mac can discover and connect
5. Other devices on the LAN **cannot** see or connect to the service

### LAN Mode (Opt-In)
1. User toggles "Allow connections from other devices" in Settings
2. `acceptLocalOnly = false` is applied
3. Listener is restarted with new parameters
4. Bonjour service advertises on all network interfaces
5. Other devices on the LAN can discover and connect (subject to approval)

### Dynamic Reconfiguration
1. Settings toggle change triggers `setAllowLANConnections()`
2. Flows through: ServerController → ServerNetworkManager → NetworkDiscoveryManager
3. `NetworkDiscoveryManager` cancels the old listener
4. Creates a new listener with updated parameters
5. Restarts on an ephemeral port
6. Existing connections remain unaffected

## Testing & Validation

### Test 1: Default Loopback-Only Operation
```bash
# On the same Mac:
dns-sd -B _mcp._tcp local.
# Expected: iMCP instance appears

# On a different Mac on the same LAN:
dns-sd -B _mcp._tcp local.
# Expected: iMCP instance does NOT appear
```

### Test 2: CLI Connection (Loopback Mode)
```bash
# On the same Mac where iMCP is running:
imcp-server
# Expected: Successfully discovers and connects
```

### Test 3: Enable LAN Access
1. Open iMCP Settings
2. Navigate to General
3. Toggle "Allow connections from other devices on this network" ON
4. Run `dns-sd -B _mcp._tcp local.` on a different Mac
5. Expected: iMCP instance now appears

### Test 4: Disable LAN Access
1. Toggle the setting OFF in Settings
2. Wait a few seconds
3. Run `dns-sd -B _mcp._tcp local.` on a different Mac
4. Expected: iMCP instance disappears

### Test 5: Persistence
1. Set "Allow LAN connections" to ON
2. Quit iMCP
3. Relaunch iMCP
4. Check `dns-sd -B _mcp._tcp local.` from another Mac
5. Expected: Setting persists, instance remains visible

### Test 6: Existing Connections
1. Enable LAN access and connect a client
2. Disable LAN access while the client is connected
3. Expected: 
   - Client connection remains active
   - New discoveries from other machines stop
   - Existing client can continue to use the connection

### Test 7: Port Conflict Recovery
1. Enable the service
2. Trigger a port conflict (start another service on the same port)
3. Check the logs for "Restarted listener with a dynamic port (allowLAN: ...)"
4. Verify the `allowLAN` setting is preserved in the log message

## Backwards Compatibility

### For Single-Mac Users (Majority)
- **No behavior change**: Service was effectively loopback-only in practice
- CLI continues to work exactly as before

### For Multi-Mac Users (Rare)
- **Breaking change**: LAN discovery stops working by default
- **Migration path**: 
  1. Users will notice connections fail
  2. Open Settings → General
  3. Enable "Allow connections from other devices"
  4. Service resumes working across machines

### For Trusted Clients
- Trusted clients list is preserved
- Applies only to clients that can still connect (respects the LAN setting)

## Code Quality

### Type Safety
- Uses Swift's strong typing throughout
- Actor isolation prevents data races

### Error Handling
- Validates discovery manager initialization
- Gracefully handles restart failures with logging
- Guards against redundant setting changes

### Logging
- Info-level logs for setting changes
- Debug logs for no-op changes
- Notice logs for listener restarts
- Error logs for failures

### Separation of Concerns
- UI layer (SettingsView) handles presentation
- Controller layer (ServerController) manages state
- Network layer (ServerNetworkManager) coordinates components
- Discovery layer (NetworkDiscoveryManager) handles low-level networking

## Future Enhancements (Out of Scope)

1. **mDNS Interface Selection**: More granular control (e.g., "Only Ethernet" or "Only Wi-Fi")
2. **Instance Naming**: Support for multiple servers with unique names (Option A)
3. **Authentication**: Require passwords for LAN connections
4. **Connection Notifications**: Alert when LAN clients connect (partially implemented for trusted clients)

## Related Issues

- Resolves the core problem in the multi-instance discovery issue
- Partially addresses security issue #103 by making loopback-only the default
- Maintains compatibility with the CLI StdioProxy implementation
- No changes needed to `CLI/main.swift` (as specified in Option B requirements)

## Migration Notes for Users

If you were using iMCP across multiple Macs on your network:

1. Update to this version
2. Open Settings → General on each Mac
3. Enable "Allow connections from other devices on this network"
4. Your cross-machine setup will resume working

The default loopback-only mode provides better security and eliminates the silent multi-instance problem for the vast majority of users who run iMCP on a single Mac.
