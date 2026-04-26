# SOP: Meta XR Simulator Test

Run a Godot project against the Meta XR Simulator on macOS, capture a screenshot
of the 2D test results panel and video of the XR view using macOS native tools only.

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

`test_main.gd` must cap FPS and use a time-based quit:
```gdscript
Engine.max_fps = 60
const QUIT_AFTER_SECONDS := 30.0
var _elapsed := 0.0

func _process(delta: float) -> void:
    _elapsed += delta
    if _elapsed >= QUIT_AFTER_SECONDS:
        get_tree().quit()
```

## 3. Capture screenshot (macOS screencapture)

```sh
GODOT=/path/to/godot.macos.editor.dev.double.arm64
PROJECT=/path/to/project

"$GODOT" --path "$PROJECT" &
GODOT_PID=$!
sleep 5

# Screenshot the full screen
screencapture -x /tmp/test-screenshot.png

# Screenshot a specific window by ID
WIN_ID=$(osascript -e \
  'tell app "System Events" to id of first window of \
   (first process whose name is "MetaXRSimulator")')
screencapture -x -l "$WIN_ID" /tmp/xr-window.png

wait $GODOT_PID
```

## 4. Capture video (macOS screencapture -V)

`screencapture -V <seconds>` records video for N seconds then saves.
Must be run while the target window is visible.

```sh
"$GODOT" --path "$PROJECT" &
GODOT_PID=$!
sleep 5

# Record 10 seconds of the full screen as video
screencapture -V 10 /tmp/test-recording.mov

wait $GODOT_PID
```

## 5. Capture video (Godot built-in MovieWriter)

For recording just the Godot viewport with no external tools:

Add to `project.godot`:
```
[movie_writer]
movie_file="res://test-output.avi"
```

Or pass on command line:
```sh
"$GODOT" --path "$PROJECT" --movie /tmp/test-output.avi
```

## 6. XR keyboard controls (Meta XR Simulator)

| Key | Action |
|-----|--------|
| W/A/S/D | Move head forward/left/back/right |
| R/F | Move head up/down |
| Arrow keys | Rotate head |
| Space | Reset controller poses |
| 2 | Poke interaction |
| 3 | Pinch |
| 4 | Grab |

Send while simulator is frontmost:
```sh
osascript -e 'tell application "System Events"
  set frontmost of (first process whose name is "MetaXRSimulator") to true
  key code 126  -- up arrow
  delay 0.3
  keystroke "2" -- poke
end tell'
```

## Pass condition

- Godot loads without script errors
- 2D panel shows PASS/FAIL rows
- XR simulator shows canvas plane
- No `File not found` in Godot stderr
