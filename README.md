<div align="center"><img width="806" height="529" alt="Hand-Scoreboard" src="https://github.com/user-attachments/assets/9129fd14-8ca1-42c5-b8c8-04697f522311" /></div>
# Hand Gesture Scoreboard

An Axis ACAP that runs a hand-gesture-driven dual scoreboard — detection and scoring in a single `.eap`, no companion app required.

Built on the open-source [DetectX](https://github.com/pandosme/DetectX) (Fred Juhlin / pandosme) with a custom Hand Gesture TFLite model and a built-in scoring engine.

**Documentation / GitHub Pages:** [https://[your-org].github.io/handgesture_scoreboard/](docs/index.html)

---

## How it works

The camera runs gesture detection on every frame. When a gesture is recognised inside a configured zone, its rule is applied to that side's score. Both scores are pushed immediately to a **CamOverlay Custom Graphics** service, so the scoreboard updates on-screen in real time.

<img width="100%" alt="hand-gestures" src="https://github.com/user-attachments/assets/f9188c9b-5ccf-4449-9a8a-b786fec637c5" />

| Zone | Side |
|------|------|
| Left half of frame | Home |
| Right half of frame | Visitors |

Zones are fully customisable — drag-to-draw in the UI.

---

## Quick Start

### Requirements

- Axis camera with **ARTPEC-8** (aarch64) and AXIS OS 11+
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) for building
- [CamOverlay](https://www.camstreamer.com/camoverlay) installed on the camera with a Custom Graphics service configured

### Build

```sh
cd handgesture_scoreboard
docker build --platform linux/arm64 -t handgesture_scoreboard .
# .eap appears in the build output — copy it out of the container
```

Or with the bundled script:

```sh
./build.sh
```

### Install

1. Camera → **System → Apps** → enable *Allow unsigned apps*
2. Upload the `.eap` — wait ~20 s for the model to load
3. Open the app settings and go to the **🎯 Scoreboard** tab

---

## Settings & Views

All configuration lives in a single settings page. Sections expand/collapse as needed.

### Scoreboard

The top panel. Shows the live Home / Visitors score and the zone editor.

- **Draw zones** over the snapshot: click *Draw Home* or *Draw Visitors*, drag a rectangle on the camera image. Zones are in 0–1000 normalised space so they work regardless of stream resolution.
- **Reset** resets both scores to their initial values.
- **Push to Overlay** re-sends the current score to CamOverlay (useful after a service restart).

### Scoring Rules

Maps each recognised gesture to a scoring action.

| Emoji | Label | Default rule | Description |
|-------|-------|-------------|-------------|
| 👍 | `like` | **+1** | Thumbs up |
| 👎 | `dislike` | **−1** | Thumbs down |
| ✊ | `fist` | **reset** | Closed fist |
| 🖐️ | `palm` | **+1** | Open palm, facing forward |
| ✋ | `stop` | **+1** | Stop — fingers together, palm forward |
| 🤚 | `stop_inverted` | **+1** | Stop — back of hand forward |
| 🤙 | `call` | — | Thumb and pinky extended |
| ✌️ | `peace` | — | V sign, palm forward |
| 🤞 | `peace_inverted` | — | V sign, back of hand forward |
| 🤘 | `rock` | — | Index and pinky extended |
| 👌 | `ok` | — | Thumb and index circle |
| ☝️ | `one` | — | One finger pointing up |
| ✌️ | `two_up` | — | Two fingers up, palm forward |
| 🤞 | `two_up_inverted` | — | Two fingers up, back of hand forward |
| 🤟 | `three` | — | Three fingers extended |
| 🖖 | `four` | — | Four fingers extended |
| 🤌 | `thumb_index` | — | Thumb and index finger |
| 🖕 | `middle_finger` | — | Middle finger |
| 🤫 | `mute` | — | Index finger to lips |
| ⬜ | `no_gesture` | — | No gesture (ignored) |

Edit any rule to `+N`, `-N`, or `reset`. Set to `0` to ignore a gesture entirely.

### CamOverlay Custom Graphics

Connects the scoreboard to a CamOverlay overlay service.

| Setting | Description |
|---------|-------------|
| Service ID | CamOverlay Custom Graphics service number (default: 20) |
| Home field | Field name in the CamOverlay widget for the Home score (default: `field1`) |
| Visitors field | Field name for the Visitors score (default: `field2`) |
| Home / Visitors colour | Text colour in RRRGGGBBB format (default: white `255255255`) |
| Show overlay gesture | Gesture that turns the overlay on (optional) |
| Hide overlay gesture | Gesture that turns the overlay off (optional) |
| Blink LED on score | Flashes the camera status LED for the cooldown duration after each score |
| Cooldown (ms) | Minimum time between successive scores in the same zone. Half this value applies as a cross-zone guard (default: 1200 ms) |

**Cooldown behaviour:**
- Same zone: must wait the full cooldown before the same gesture scores again.
- Other zone: must wait half the cooldown — prevents accidental cross-zone triggers while moving a hand between zones.

### Games / Sets

Optional win-tracking layer on top of the point score.

- Enable to count games/sets.
- When a side reaches the **target points** (default: 11) and leads by at least **win-by** (default: 2), it wins a game — points reset, game count increments.
- Game counts are pushed to two additional CamOverlay fields (default: `field3` / `field4`).

### Live Detection View

Expandable live preview showing the camera stream with gesture bounding boxes overlaid in real time. Use this to verify detection quality and zone placement without leaving the settings page.

Inside this panel, two tabs control the detection geometry:

- **Area of Interest** — draw a polygon; only gestures detected inside it are processed.
- **Exclude Areas** — draw one or more polygons to suppress detections in specific parts of the frame.

### Detection Settings

| Setting | Description |
|---------|-------------|
| Confidence threshold | Minimum model confidence (0–100) to accept a detection |
| Active labels | Toggle individual gestures on/off |
| Prioritize | *Accuracy* suppresses false triggers; *Responsiveness* reduces latency |
| Stabilize transition | Minimum frames a gesture must persist before its event fires |

### SD Card Training Capture

When enabled, saves full-frame JPEG images and YOLO-format label files to the SD card whenever a gesture is detected. Useful for collecting site-specific data to fine-tune the model.

- Configurable capture interval (1–60 s)
- **Download Archive** — packages all captured images and labels into a zip
- **Clear All** — removes stored files
- Auto-stops at 2 000 images

### MQTT

Optional MQTT output for integration with external systems.

| Topic | Payload |
|-------|---------|
| `detectx/detection/<serial>` | All bounding boxes, labels, confidence each frame |
| `detectx/events/<serial>/<label>/true\|false` | State-change events per gesture |
| `detectx/crops/<serial>` | Cropped detection image (base64) + metadata |

Configure broker address, port, optional username/password, and a pre-topic prefix.

### Detection Export (Cropping)

When downstream systems need cropped images of each detected gesture:

- Enable/disable cropping
- Set border padding around each crop (presets: None, 25 px, 50 px, 100 px)
- Output via MQTT or HTTP POST
- Throttle output frequency
- **View Latest Crops** — gallery of the 10 most recent crops for quality checking

### Model

Upload a replacement `.tflite` model and `labels.txt` directly from the browser — no SSH or Docker rebuild required.

### About

Live dashboard: application name/version/vendor, device model/firmware/serial/CPU load, model status/inference time/DLPU, and MQTT broker status.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Overlay not updating | Verify CamOverlay service ID and field names match; check *CamOverlay* status chip on the Scoreboard panel |
| Score fires multiple times per gesture | Increase the cooldown value |
| Score fires across zones accidentally | Half-cooldown cross-zone guard is active; increase overall cooldown if needed |
| 503 errors in browser console | ACAP binary crashed — reinstall or restart the app |
| Camera restarts on app stop | Expected on hard crash; LED blink now uses param.cgi so this should not happen in normal use |

Logs: `ssh root@<camera-ip> "tail -f /var/log/syslog | grep Scoreboard"`

---

## Version

| Version | Notes |
|---------|-------|
| 1.0.0 | Initial release — single-app scoreboard, CamOverlay push, zones, cooldown, LED blink, Games/Sets |

Based on [DetectX](https://github.com/pandosme/DetectX) 4.1.0 by Fred Juhlin.
