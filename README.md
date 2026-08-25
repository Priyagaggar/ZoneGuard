# ZoneGuard — Multi-Zone Intrusion Detector

Detects people/vehicles entering or exiting user-defined polygon zones in a video, using YOLOv8 for detection, SORT for tracking, and polygon collision tests for zone logic. Events are logged with timestamp, object ID, and zone.

See [DECISIONS.md](DECISIONS.md) for why each piece was chosen, [PROGRESS.md](PROGRESS.md) for current status, and [ISSUES_AND_LEARNINGS.md](ISSUES_AND_LEARNINGS.md) for bugs/fixes along the way.

## How it works

Each video frame flows through the same pipeline:

1. **Frame** is read from the video file.
2. **YOLOv8 detection** (`detector.py`) finds people/vehicles in the frame, each with a box and confidence score.
3. **SORT tracking** (`sort_tracker.py`) matches detections to existing tracks frame-to-frame (Kalman filter + Hungarian/IoU matching) and assigns a stable object ID that persists across frames.
4. **Zone polygon check** (`zone_logic.py`) tests each tracked object's box center against every zone with `cv2.pointPolygonTest`, and compares against its membership last frame to detect a crossing.
5. **Event logging** (`event_logger.py`) records any enter/exit crossing with a timestamp, object ID, and zone name — to console, on-screen overlay, and `logs/events.csv`.

## Setup

```
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

First run of `main.py` or `zone_drawer.py` will auto-download YOLOv8 weights (`yolov8n.pt`, ~6MB) via ultralytics.

## Usage

### 1. Draw zones on your video

```
python zone_drawer.py --video path\to\video.mp4 --zones zones.json
```

Controls:
- Left click — add a polygon point
- Right click — undo last point
- Enter / `c` — close the polygon, then type a zone name in the console
- `s` — save all zones to `zones.json`
- `r` — clear all zones
- `q` / Esc — quit

Draw as many zones as you like before saving. Zones persist in `zones.json`.

### 2. Run detection + tracking + intrusion logging

```
python main.py --video path\to\video.mp4 --zones zones.json
```

- Detected people/vehicles are boxed and tracked with a stable ID.
- An object's box turns red while its center point is inside any zone.
- Enter/exit events print to console, overlay on the video, and are appended to `logs/events.csv` (columns: `timestamp, object_id, zone, event`).
- Press `q` in the video window to stop.

Run with `--no-display` for headless operation (e.g. batch processing).

Run with `--debug` to log per-frame YOLO confidence/class for every raw detection, the active SORT config, and every new/lost-track and in-zone observation to `logs/debug.csv` (fresh file each run) plus console — useful for diagnosing spurious track IDs or unexpected zone events.

## Known limitations

- **New ID after long occlusion.** If a tracked object is occluded or undetected for longer than `max_age` (15 frames, `config.py`), its track is evicted and it gets a brand-new ID on reappearance rather than resuming its old one. Plain SORT has no appearance-based re-identification to recognize "this is the same object I saw before" after a long gap — Deep SORT (which adds appearance embeddings) would fix this, but was deliberately not chosen for this project (see DECISIONS.md).
- **Boundary flicker.** An object standing right at a zone edge can trigger rapid enter/exit events from small, real movements straddling the line. This is correct behavior given the current point-in-polygon check, not a bug — a hysteresis margin around zone edges would reduce it if it becomes a problem.

Full evidence trace for how these were diagnosed (and the three related bugs that were fixed along the way) is in [ISSUES_AND_LEARNINGS.md](ISSUES_AND_LEARNINGS.md).

## Project layout

| File | Purpose |
|---|---|
| `config.py` | Central tunables (thresholds, colors, file paths) |
| `zone_drawer.py` | Interactive zone-drawing tool, saves `zones.json` |
| `zone_logic.py` | `Zone`/`ZoneManager` — polygon tests + enter/exit event detection |
| `detector.py` | YOLOv8 wrapper, filters to person/vehicle classes |
| `sort_tracker.py` | From-scratch SORT (Kalman filter + Hungarian matching) tracker |
| `event_logger.py` | CSV logging + in-memory feed for on-screen overlay |
| `main.py` | Ties it all together: video loop -> detect -> track -> zone check -> log -> display |
