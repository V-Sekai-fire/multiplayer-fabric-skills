# SOP: Meta XR Simulator Test

Run a Godot project against the Meta XR Simulator on macOS, capture screenshots
of the 2D test results and XR view using window ID detection (no hardcoded coordinates).

## Paths

| Resource | Path (relative to monorepo root) |
|---|---|
| Godot binary | `multiplayer-fabric-godot/bin/godot.macos.editor.dev.double.arm64` |
| Project | `multiplayer-fabric-interaction-system-project/` |
| Simulator app | `multiplayer-fabric-meta/MetaXRSimulator.app` |
| Capture script | `artifacts/capture_windows.sh` |
| Artifacts out | `artifacts/` |

Monorepo root: `/Users/ernest.lee/Downloads/multiplayer-fabric/`

## 1. Verify simulator is activated

```sh
cat /usr/local/share/openxr/1/active_runtime.json | grep name
# expected: "Meta OpenXR Simulator"
```

If not active:
```sh
sudo bash /Users/ernest.lee/Downloads/multiplayer-fabric/multiplayer-fabric-meta/MetaXRSimulator.app/Contents/Resources/MetaXRSimulator/activate_simulator.sh
```

## 2. Ensure simulator is running

```sh
pgrep -l MetaXRSimulator || open /Users/ernest.lee/Downloads/multiplayer-fabric/multiplayer-fabric-meta/MetaXRSimulator.app
# wait ~3s if just launched
```

## 3. Project requirements

`project.godot` must have:
```
[xr]
openxr/enabled=true
```

`test_main.gd` must:
- Cap FPS: `Engine.max_fps = 60`
- Use time-based quit: `QUIT_AFTER_SECONDS := 30.0`
- Stamp window title with a ULID for unique run identification:
  ```gdscript
  DisplayServer.window_set_title("My Test [%s]" % _gen_ulid())
  ```

## 4. Launch Godot (background)

```sh
GODOT=/Users/ernest.lee/Downloads/multiplayer-fabric/multiplayer-fabric-godot/bin/godot.macos.editor.dev.double.arm64
PROJECT=/Users/ernest.lee/Downloads/multiplayer-fabric/multiplayer-fabric-interaction-system-project
"$GODOT" --path "$PROJECT" &
```

**Do NOT** pass `--quit-after N` — that counts frames, not seconds. The project's
own `QUIT_AFTER_SECONDS` timer handles shutdown.

## 5. Wait for XR session to initialise, then capture

```sh
sleep 8
GODOT_PID=$(pgrep "godot.macos.edi" | head -1)
bash /Users/ernest.lee/Downloads/multiplayer-fabric/artifacts/capture_windows.sh \
  "$GODOT_PID" \
  /Users/ernest.lee/Downloads/multiplayer-fabric/artifacts
```

Output:
```
ULID=<id>  Godot wid=<n>  Sim wid=<n>
Saved <ULID>-godot.png
Saved <ULID>-xr.png
```

## capture_windows.sh internals

```bash
#!/bin/bash
# Usage: capture_windows.sh <godot_pid> <out_dir>
GODOT_PID=$1
OUT_DIR=${2:-/tmp}

ULID=$(python3 -c "
import time, random
chars = '0123456789ABCDEFGHJKMNPQRSTVWXYZ'
t = int(time.time() * 1000)
result = ''
for i in range(10):
    result = chars[t & 0x1F] + result
    t >>= 5
for i in range(16):
    result += chars[random.randint(0, 31)]
print(result)
")

WINDOWS=$(swift -e '
import CoreGraphics
let list = CGWindowListCopyWindowInfo([.optionOnScreenOnly, .excludeDesktopElements], kCGNullWindowID) as! [[String:Any]]
for w in list {
  let owner = (w[kCGWindowOwnerName as String] as? String) ?? ""
  let wid   = (w[kCGWindowNumber as String] as? Int) ?? 0
  if owner.lowercased().contains("godot") { print("godot \(wid)") }
  if owner == "MetaXRSimulator" { print("sim \(wid)") }
}
' 2>/dev/null)

GODOT_WID=$(echo "$WINDOWS" | grep "^godot" | awk '{print $2}' | head -1)
SIM_WID=$(echo "$WINDOWS"   | grep "^sim"   | awk '{print $2}' | head -1)

if [ -n "$GODOT_WID" ]; then
  screencapture -x -l "$GODOT_WID" "$OUT_DIR/${ULID}-godot.png"
fi
if [ -n "$SIM_WID" ]; then
  screencapture -x -l "$SIM_WID" "$OUT_DIR/${ULID}-xr.png"
fi
```

Key points:
- `CGWindowListCopyWindowInfo` via Swift — no hardcoded coordinates
- `screencapture -x -l <wid>` captures a specific window by CGWindow ID regardless of position
- ULID names artifacts for sortable, unique identification

## 6. XR keyboard controls (Meta XR Simulator)

| Key | Action |
|-----|--------|
| W/A/S/D | Move head forward/left/back/right |
| R/F | Move head up/down |
| Arrow keys | Rotate head (tilt up = key code 126) |
| Space | Reset controller poses |
| 2 | Poke interaction |
| 3 | Pinch |
| 4 | Grab |

Send while simulator is frontmost:
```sh
osascript -e 'tell application "System Events"
  set frontmost of (first process whose name is "MetaXRSimulator") to true
  delay 0.3
  key code 126  -- up arrow: tilt head up
  delay 0.3
  key code 126
end tell'
```

## 7. S2H debug sky shader

All debug info (ULID, head/controller positions, distance to canvas, elapsed time)
renders in `debug_sky.gdshader` via `shader_type sky` + GLSL preprocessor:
- `s2h_drawSkybox(ctx)` — world grid + horizon
- `ContextGather` text overlay composited over sky colour

Key: sky shaders use the GLSL preprocessor (supports `#include`). For
spatial/canvas_item shaders, `s2h.gdshaderinc` requires `g_miniFont[192]` (explicit
size) — unsized `g_miniFont[]` fails GDShader tokenizer.

## 8. Capture video (screencapture native)

```sh
# Record N seconds — interactive, click to stop early
screencapture -V 10 recording.mov

# Or use Godot's built-in MovieWriter (no external tool)
# godot --movie /tmp/out.avi
```

## Pass condition

- Both `<ULID>-godot.png` and `<ULID>-xr.png` saved in `artifacts/`
- Godot window title shows a unique ULID each run
- `godot-window.png` shows PASS/FAIL test result rows
- `xr-window.png` shows S2H grid sky + ULID text overlay + canvas plane with test UI
- No `File not found` errors in Godot stderr
