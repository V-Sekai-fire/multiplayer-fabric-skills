# SOP: Meta XR Simulator Test

Run a Godot project against the Meta XR Simulator on macOS, capture screenshots
of the 2D test results and XR view using window ID detection (no hardcoded coordinates).

## Prerequisites

- Meta XR Simulator activated as the OpenXR runtime
- Godot binary available
- Test project at a known path

## 1. Verify simulator is activated

```sh
cat /usr/local/share/openxr/1/active_runtime.json | grep name
# expected: "Meta OpenXR Simulator"
```

If not active:
```sh
sudo bash /path/to/MetaXRSimulator.app/Contents/Resources/MetaXRSimulator/activate_simulator.sh
```

## 2. Project requirements

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

## 3. Capture windows by CGWindow ID (no hardcoded coords)

Use Swift to get CGWindow IDs, then `screencapture -l <wid>` to capture each window independently of position:

```sh
# Get window IDs
WINDOWS=$(swift -e '
import CoreGraphics
let list = CGWindowListCopyWindowInfo([.optionOnScreenOnly,.excludeDesktopElements], kCGNullWindowID) as! [[String:Any]]
for w in list {
  let owner = (w[kCGWindowOwnerName as String] as? String) ?? ""
  let wid   = (w[kCGWindowNumber as String] as? Int) ?? 0
  if owner.lowercased().contains("godot") { print("godot \(wid)") }
  if owner == "MetaXRSimulator" { print("sim \(wid)") }
}
' 2>/dev/null)

GODOT_WID=$(echo "$WINDOWS" | grep "^godot" | awk '{print $2}' | head -1)
SIM_WID=$(echo "$WINDOWS"   | grep "^sim"   | awk '{print $2}' | head -1)

screencapture -x -l "$GODOT_WID" godot-window.png
screencapture -x -l "$SIM_WID"   xr-window.png
```

## 4. XR keyboard controls (Meta XR Simulator)

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

## 5. Ego-centric headlamp

Parent `Label3D` nodes to the `XRCamera3D` at fixed camera-space offsets. They dip
into the lower-left corner of the XR viewport showing world-centric / ego-centric
axis names (RIGHT/MODEL_LEFT, UP/MODEL_TOP, BACK/MODEL_FRONT).

## 6. World-centric debug sky

Use `s2h_drawSkybox()` from `godot-ShaderToHuman` in a `sky` shader type.
`POSITION` (camera world pos) is available in sky shaders as the ray origin.

## 7. Capture video (screencapture native)

```sh
# Record N seconds — interactive, click to stop early
screencapture -V 10 recording.mov

# Or use Godot's built-in MovieWriter (no external tool)
# Add to project.godot: [movie_writer] movie_file="res://out.avi"
# Or pass: godot --movie /tmp/out.avi
```

## Pass condition

- Godot window title shows a unique ULID each run
- `godot-window.png` shows PASS/FAIL test result rows
- `xr-window.png` shows S2H grid sky + ego headlamp labels + canvas plane
- No `File not found` errors in Godot stderr
