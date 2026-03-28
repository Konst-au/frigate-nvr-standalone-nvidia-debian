# Frigate NVR — Self-Hosted AI Security Camera System

## Production Setup on Debian 12 with NVIDIA GPU, YOLOv9c ONNX, and Overhead Camera Workarounds

A complete, battle-tested Frigate NVR deployment including export automation, health monitoring, Google Drive sync, and a hard-won solution for overhead/downward-angle cameras that COCO-trained models refuse to acknowledge as containing humans.

---

## Hardware

| Component | Spec |
| --- | --- |
| CPU | Intel i7-4790K |
| RAM | 32GB DDR3 |
| GPU | NVIDIA GeForce GTX 1650 4GB |
| OS Drive | 500GB SSD |
| Storage | 2×1TB HDD RAID1 (mdadm) |
| OS  | Debian 12 |
| Cameras | 3× HiLook IPC-T269H (6MP, HiLook by Hikvision) |

---

## The Journey (abbreviated for your sanity)

This setup took approximately 10 days, two full system restores from Rescuezilla, three model switches, several hundred megabytes of RAM consumed by ill-advised local ONNX export attempts, and one $30 Claude AI subscription that turned out to be the most cost-effective debugging tool ever purchased.

### Models tried, in order of suffering:

1. **YOLO11x** — Downloaded. Crashed Frigate with CUDA graph errors. Incompatible attention layers. Binned.
2. **YOLO11L** — Worked. Ran at 94% GPU util. Completely blind to overhead humans. Produced ominous `shape massaging nodes will execute on CPU` warning on every startup. Replaced.
3. **YOLOv9c** — Current. Recommended by Frigate docs. Better overhead detection. Still needed the rotation hack for the NW camera.

### The OOM Incident

At some point, someone attempted to export a YOLO model locally using torch/ultralytics. The 32GB RAM system ran out of memory mid-export, triggering a kernel OOM panic, corrupting the running system, and necessitating a full restore from Rescuezilla.

**LESSON: Never export ONNX models locally. Always download pre-built from HuggingFace.**

```
https://huggingface.co/deepghs/yolos/
```

### The Overhead Camera Problem

The NW Side camera is mounted overhead looking straight down into a side passage. Every COCO-trained YOLO model has been trained on people standing upright at eye level. When you appear as a horizontal blob from above in IR greyscale at night, the model's response is: "I see some pixels. Not my problem."

**The fix:** Rotate the detect stream 270 degrees in go2rtc before feeding it to Frigate.

```yaml
go2rtc:
  streams:
    nw_side_detect:
      - "ffmpeg:nw_side_2#video=h264#rotate=270"
```

Note: `rotate=90` shows you upside down. `rotate=270` is correct. This distinction cost approximately 45 minutes.

---

## Architecture

```
Camera (RTSP) → go2rtc → Frigate (ONNX/CUDA) → SQLite DB
                                ↓
                    Recording segments (RAID)
                                ↓
                    frigate_export.py (every 5min via systemd)
                                ↓
                    /clips/export/*.mp4 (rotated, named)
                                ↓
                    rclone → Google Drive (SecurityClips)
```

---

## Detection Model

**YOLOv9c**, pre-built ONNX from HuggingFace:

```bash
wget https://huggingface.co/deepghs/yolos/resolve/main/yolov9c/model.onnx \
  -O ~/frigate/config/model_cache/yolov9c-640.onnx
```

Config:

```yaml
model:
  model_type: yolo-generic
  width: 640
  height: 640
  input_tensor: nchw
  input_dtype: float
  path: /config/model_cache/yolov9c-640.onnx
  labelmap_path: /labelmap/coco-80.txt

detectors:
  onnx:
    type: onnx
    cuda_graphs: true
```

`cuda_graphs: true` works with YOLOv9c. Does NOT work with YOLO11 variants.

---

## Camera Configuration

### Standard cameras

```yaml
  front:
    ffmpeg:
      inputs:
        - path: rtsp://user:pass@192.168.1.44:554/Streaming/Channels/101
          roles: [record, audio]
        - path: rtsp://user:pass@192.168.1.44:554/Streaming/Channels/102
          roles: [detect]
    detect:
      width: 640
      height: 360
      fps: 8
```

### Overhead camera with rotation

```yaml
go2rtc:
  streams:
    nw_side_2:
      - rtsp://user:pass@192.168.1.46:554/Streaming/Channels/102
    nw_side_detect:
      - "ffmpeg:nw_side_2#video=h264#rotate=270"

cameras:
  nw_side:
    ffmpeg:
      inputs:
        - path: rtsp://user:pass@192.168.1.46:554/Streaming/Channels/101
          roles: [record, audio]
        - path: rtsp://127.0.0.1:8554/nw_side_detect
          roles: [detect]
    detect:
      width: 360
      height: 640
      fps: 8
```

Width/height are swapped because the stream is portrait after rotation.

---

## Object Detection Tuning

```yaml
objects:
  filters:
    person:
      min_area: 1000
      min_score: 0.4
      threshold: 0.55
    car:
      min_area: 3000
      min_score: 0.4
      threshold: 0.5
```

`threshold` is the median score across multiple frames needed to confirm a true positive. Default is 0.7 — too high for overhead/IR scenarios. 0.55 is a good balance.

---

## Export Script Key Settings

```python
SEGMENTS_BEFORE    = 1      # ~10s pre-event buffer
SEGMENTS_AFTER     = 1      # ~10s post-event buffer
MERGE_GAP_SEC      = 90     # merge events within 90s
MIN_DURATION_SEC   = 1      # minimum event duration to export
LOOKBACK_HOURS     = 24
EXPORT_RETAIN_DAYS = 60
```

Filename format: `2026-03-26__08.01.02_front_person.mp4`

---

## Known Issues / Lessons Learned

1. Never run local torch ONNX export — will OOM kill your system
2. cuda_graphs: only works with models without attention layers (YOLOv9, not YOLO11)
3. Overhead cameras need stream rotation — rotate=270 for correct orientation
4. rclone staleness check — use systemd ExecMainExitTimestamp, not log file mtime
5. Camera rename — update DB paths AND folder names AND all config references
6. YOLO11 on overhead IR = expensive paperweight for that specific use case

---

## Cost

- Hardware: already owned
- Software: all free/open source
- AI assistance: $34 Claude Pro subscription

Total: **$34** and approximately 10 evenings of your life.

I posted the battle on my Facebook as well: https://www.facebook.com/share/p/17pmERFHv2/ 
