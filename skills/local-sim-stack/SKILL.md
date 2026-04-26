---
name: local-sim-stack
description: Run the multiplayer-fabric Quest 3 smoke test locally on macOS using Meta XR Simulator v85 + Godot. Starts the zone server (headless) then the observer scene (OpenXR client) and verifies connection. Use when iterating on the smoke test or validating a new Godot build.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Local Meta XR Simulator smoke test stack

Minimum smoke test pass condition: Godot observer scene initialises an OpenXR
session via Meta XR Simulator, connects to the zone server on `127.0.0.1:7443`,
and the simulator shows a multiplayer session in its side panel.

## Prerequisites

| Component | Path |
|---|---|
| Meta XR Simulator v85 | `multiplayer-fabric-meta/MetaXRSimulator.app` |
| Godot editor binary | `multiplayer-fabric-godot/bin/godot.macos.editor.dev.arm64` |
| Game project | `multiplayer-fabric-abyssal/` |
| CRDB certs | `multiplayer-fabric-hosting/certs/crdb/` |

## 1. Activate the OpenXR runtime

Run once per machine (or after a macOS update):

```sh
sudo bash multiplayer-fabric-meta/MetaXRSimulator.app/Contents/Resources/MetaXRSimulator/activate_simulator.sh
```

Verify:

```sh
cat /usr/local/share/openxr/1/active_runtime.json
# Must contain "Meta"
```

Or use the Elixir script (records result to CRDB):

```sh
cd multiplayer-fabric-meta && elixir activate_smoke_test.exs
```

## 2. Launch Meta XR Simulator

```sh
open multiplayer-fabric-meta/MetaXRSimulator.app
```

The activation toggle in the top-left title bar must be **on** (slider right).
The window shows the synthetic environment ("ZONE 2" label is the default scene).

### Accessibility note — macOS toggle

The global activation toggle is a custom `UI element` (not a standard button).
Click it via osascript if the UI is not responding:

```sh
osascript << 'EOF'
tell application "System Events"
  tell process "MetaXRSimulator"
    tell window 1
      set allEls to every UI element
      click item 3 of allEls   -- position 2061,279 size 26x15 = the toggle knob
    end tell
  end tell
end tell
EOF
```

If that fails, run `activate_simulator.sh` with sudo directly (see step 1).

## 3. Stop the Docker zone server (if running)

The Docker image `ghcr.io/v-sekai-fire/zone-fabric:latest` is `amd64`-only and
crashes on Apple Silicon (Godot Mono build, missing .NET assemblies). It binds
UDP 7443 but cannot serve connections. Stop it before running locally:

```sh
docker stop multiplayer-fabric-hosting-zone-server-1 2>/dev/null || true
```

Verify port 7443 is free:

```sh
lsof -i udp:7443
# Should show nothing (or only your Godot process after step 4)
```

## 4. Start the local Godot zone server

```sh
GODOT=multiplayer-fabric-godot/bin/godot.macos.editor.dev.arm64
ABYSSAL=multiplayer-fabric-abyssal

"$GODOT" --headless --path "$ABYSSAL" --scene scenes/zone_server.tscn > /tmp/zone_server.log 2>&1 &
echo "Zone server PID: $!"
```

Wait ~3 s then confirm it is listening:

```sh
lsof -i udp:7443   # must show godot.macos process
grep "cert_hash" /tmp/zone_server.log
```

Expected:
```
{"cert_hash":"<base64>","event":"ready","port":7443}
```

## 5. Launch the observer scene (OpenXR client)

```sh
"$GODOT" --path "$ABYSSAL" --scene scenes/observer.tscn > /tmp/observer.log 2>&1 &
```

## 6. Verify pass condition

After ~8 s:

```sh
grep -E "(OpenXR initialised|connecting|FabricPlayerXR)" /tmp/observer.log
grep "recv" /tmp/zone_server.log | wc -l   # should be > 0
```

Expected observer log:

```
OpenXR: Created instance for OpenXR 1.1.54
FabricClient: connecting to 127.0.0.1:7443
FabricPlayerXR: OpenXR initialised
```

Expected zone server log (datagrams arriving):

```
{"event":"recv","len":100}
{"event":"recv","len":100}
...
```

The Meta XR Simulator viewport switches from the default outdoor scene to
**Godot's rendered environment** (slate walls, terracotta floor). The
**Multiplayer Sessions** side panel shows the active session.

### Screenshot the simulator

```sh
screencapture -R 1254,249,1024,768 /tmp/meta_xr_check.png
```

Get current window position if it has moved:

```sh
osascript -e 'tell app "System Events" to tell process "MetaXRSimulator" to get (position of window 1) & (size of window 1)'
```

## 7. Teardown

```sh
pkill -f "godot.macos.editor.dev.arm64"
```

## Known issues

| Issue | Notes |
|---|---|
| `MVKBlockObserver` duplicate class warning | Harmless — MoltenVK symbol collision between Godot and SIMULATOR.so |
| `VK_ERROR_EXTENSION_NOT_PRESENT: VK_KHR_portability_enumeration` | Harmless on Apple Silicon — Metal path still works |
| Activation toggle hard to click via accessibility | Use `activate_simulator.sh` with sudo as fallback |
| `--headless` zone server also initialises OpenXR | Expected — runtime loads but submits no frames |
| Docker zone server (`zone-fabric:latest`) crashes on Apple Silicon | `amd64`-only image; Godot Mono missing .NET assemblies — stop the container and use local Godot zone server instead |
| `WebTransportPeer.create_server()` "server already listening" | `FabricMultiplayerPeer.create_server()` calls factory 4× for channel ports — use `WebTransportPeer` directly in `zone_server.tscn` instead |

## Reference

- `multiplayer-fabric-meta/activate_smoke_test.exs` — Elixir/CRDB activation script
- `multiplayer-fabric-meta/MetaXRSimulator.app/Contents/Resources/MetaXRSimulator/activate_simulator.sh`
- `multiplayer-fabric-abyssal/scenes/observer.tscn` — XR observer scene
- `multiplayer-fabric-abyssal/scenes/zone_server.tscn` — headless zone server scene
- `multiplayer-fabric-abyssal/scripts/fabric_server.gd` — WebTransportPeer server script
