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

## 3. Start the zone server (headless)

```sh
GODOT=multiplayer-fabric-godot/bin/godot.macos.editor.dev.arm64
ABYSSAL=multiplayer-fabric-abyssal

"$GODOT" --headless --path "$ABYSSAL" --scene scenes/main.tscn > /tmp/zone_server.log 2>&1 &
echo "Zone server PID: $!"
```

Wait ~3 s then confirm it is listening:

```sh
grep -E "(QUIC|transport parameter|zone_curtain)" /tmp/zone_server.log | head -5
```

Expected:
```
[zone_curtain] zone=0 bake surfaces=1
[zone_curtain] zone=1 bake surfaces=1
92c2b71d: Sending transport parameter TLS extension ...
```

## 4. Launch the observer scene (OpenXR client)

```sh
"$GODOT" --path "$ABYSSAL" --scene scenes/observer.tscn > /tmp/observer.log 2>&1 &
```

## 5. Verify pass condition

After ~5 s:

```sh
grep -E "(OpenXR initialised|connecting|FabricPlayerXR)" /tmp/observer.log
```

Expected:

```
OpenXR: Created instance for OpenXR 1.1.54
FabricClient: connecting to 127.0.0.1:7443
FabricPlayerXR: OpenXR initialised
```

The Meta XR Simulator window shows **"Multiplayer Sessions"** in its side panel
with the current session listed as **Alpha (Current)**. Additional Godot
instances appear as Charlie, Delta, etc.

### Screenshot the simulator

```sh
screencapture -R 1254,249,1024,768 /tmp/meta_xr_check.png
```

Get current window position if it has moved:

```sh
osascript -e 'tell app "System Events" to tell process "MetaXRSimulator" to get (position of window 1) & (size of window 1)'
```

## 6. Teardown

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

## Reference

- `multiplayer-fabric-meta/activate_smoke_test.exs` — Elixir/CRDB activation script
- `multiplayer-fabric-meta/MetaXRSimulator.app/Contents/Resources/MetaXRSimulator/activate_simulator.sh`
- `multiplayer-fabric-abyssal/scenes/observer.tscn` — XR observer scene
- `multiplayer-fabric-abyssal/scenes/main.tscn` — zone server scene
