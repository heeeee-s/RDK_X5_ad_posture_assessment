# Adolescent Posture Health Intelligent Screening System

> An embedded posture screening system built on the Horizon RDK X5 and a 132GS binocular depth camera.
> It fuses ROS2 depth vision nodes, 2D skeleton keypoint algorithms, and 3D point cloud analysis to quantify 5 posture indicators in millimeters within 30 seconds, then automatically generates an HTML report accessible from any phone on the local network.

---

## Repository Structure

This repository contains three sub-directories that together form the complete system:

| Directory | Type | Description |
|-----------|------|-------------|
| `hobot_stereonet/` | ROS2 package | Horizon binocular depth estimation node — outputs a 16UC1 depth map (mm) |
| `mono2d_body_detection/` | ROS2 package | Horizon on-device skeleton keypoint detection node — outputs 17 body keypoints |
| `ad_posture_assessment/` | Application | Posture analysis algorithms, scoring engine, AI advice, report generation, and Windows GUI controller |

---

## Features

- **5 posture indicators**: shoulder height imbalance, kyphosis, head tilt, pelvic tilt, and rounded shoulders — all reported as millimeter-level measurements
- **Dual-path fusion**: 2D keypoint method (shoulder / head / pelvis / rounded shoulder) + 3D point cloud layer analysis (kyphosis)
- **On-device inference**: skeleton detection runs entirely on the RDK X5 BPU — no cloud compute required
- **AI health advice**: calls the Tongyi Qianwen (Qwen) LLM API for personalized suggestions; falls back to 18 local rule templates when offline
- **HTML report**: posture score, depth image snapshots, historical trend chart, and AI advice — viewable by scanning a QR code on any phone
- **GUI controller**: Windows Tkinter app that launches all three algorithm processes on the RDK X5 via SSH with a single click
- **History records**: SQLite local database storing the last 5 sessions per user

---

## System Architecture

```
Windows PC
└── ad_posture_assessment/gui_controller.py  (Tkinter + paramiko SSH)
    └── One-click process control / live log filtering / report QR code

          │  SSH commands / log stream
          ▼

RDK X5 Board  (tROS Humble)
│
├── hobot_stereonet/                      ← ROS2 package
│   └── hobot_stereonet node
│         Input : raw images from 132GS binocular camera
│         Output: /StereoNetNode/stereonet_depth  (16UC1 depth map, mm)
│                 /StereoNetNode/rectify_left_image  (rectified left RGB)
│
├── mono2d_body_detection/                ← ROS2 package
│   └── mono2d_body_detection node
│         Input : left RGB image
│         Output: /hobot_mono2d_body_detection  (17 keypoints + confidence)
│
└── ad_posture_assessment/                  ← Application
    ├── cloud_analysis/
    │   ├── CloudPostureNode              keypoint depth projection
    │   │                                 (shoulder / head / pelvis / rounded shoulder)
    │   └── PointCloudProcessor           ROI point cloud layering (kyphosis)
    │       └── PostureResult  (5 mm measurements + status labels + scores)
    └── src/
        ├── calc_status_and_score()       weighted 0–100 composite score
        ├── PostureCoach                  AI advice (Qwen API / local templates)
        ├── database.py                   SQLite history
        └── report_generator.py           HTML report + HTTP server on :8080
```

---

## Directory Structure

```
源码/
├── hobot_stereonet/                # Horizon depth estimation ROS2 package
│   ├── config/                     # Depth network model files (.bin / .hbm)
│   ├── script/
│   │   ├── run_stereo.sh           # Launch the depth camera node
│   │   └── depth_to_pcd/           # Depth-to-point-cloud utility scripts
│   └── package.xml
│
├── mono2d_body_detection/          # Horizon skeleton detection ROS2 package
│   ├── config/                     # Detection model config files
│   ├── launch/
│   │   └── mono2d_from_stereonet.launch.py   # Launch detection from depth camera
│   ├── script/                     # Keypoint filter and auxiliary scripts
│   └── package.xml
│
└── ad_posture_assessment/            # Posture analysis application
    ├── main_cloud_rdkx5.py         # Board-side main entry point
    ├── gui_controller.py           # Windows GUI controller
    ├── requirements.txt            # Board-side Python dependencies
    ├── .env.example                # Environment variable template
    ├── remote_runner.sh            # Remote one-click launch script
    ├── cloud_analysis/             # Core algorithm package
    │   ├── __init__.py             # analyze_posture_from_cloud()
    │   ├── posture_from_cloud.py   # CloudPostureNode (ROS2 + keypoint projection)
    │   └── pointcloud_processor.py # Point cloud kyphosis algorithm
    ├── config/                     # Keypoint model configs
    └── src/
        ├── interfaces.py           # PostureResult dataclass
        ├── llm_coach.py            # AI advice engine
        ├── database.py             # SQLite database module
        └── report_generator.py     # HTML report generator
```

---

## Requirements

### RDK X5 Board

| Item | Requirement |
|------|-------------|
| OS | Ubuntu 22.04 (Horizon tROS image) |
| ROS | tROS Humble |
| Python | 3.10+ |
| Camera | 132GS binocular depth camera |

```bash
pip install dashscope python-dotenv jinja2 qrcode opencv-python
```

### Windows PC (GUI Controller)

| Item | Requirement |
|------|-------------|
| OS | Windows 10 / 11 |
| Python | 3.8+ |

```bash
pip install paramiko qrcode pillow
```

---

## Quick Start

### 1. Configure environment variables

```bash
cp ad_posture_assessment/.env.example ad_posture_assessment/templates/.env
# Edit .env and set:
# DASHSCOPE_API_KEY=your_api_key_here   (optional — falls back to local templates)
```

### 2. Configure the GUI controller

Edit the `CONFIG` block at the top of `ad_posture_assessment/gui_controller.py`:

```python
CONFIG = {
    "host":        "192.168.xxx.xxx",   # LAN IP of the RDK X5
    "port":        22,
    "username":    "root",
    "password":    "your_password",
    "project_dir": "/ad_posture_assessment",
    "report_port": 8080,
}
```

### 3. Launch via GUI (recommended)

Run the GUI controller on Windows:

```bash
python ad_posture_assessment/gui_controller.py
```

Click **Connect** → fill in the subject's name, age, and gender → click **Start All**.  
The system automatically launches in sequence:

1. `hobot_stereonet` depth camera node
2. `mono2d_body_detection` skeleton detection node
3. `ad_posture_assessment` posture analysis main program

When detection is complete, a QR code appears on the right panel. Scan it with a phone on the same network to view the full HTML report.

### 4. Manual launch (board terminal)

```bash
# Terminal 1 — depth camera
source /opt/tros/humble/setup.bash
bash hobot_stereonet/script/run_stereo.sh

# Terminal 2 — skeleton detection
source /opt/tros/humble/setup.bash
ros2 launch mono2d_body_detection mono2d_from_stereonet.launch.py

# Terminal 3 — posture analysis
source /opt/tros/humble/setup.bash
cd ad_posture_assessment
python3 main_cloud_rdkx5.py --name "Zhang San" --age 16 --gender male
```

---

## Scoring Reference

### Per-indicator thresholds (unit: mm)

| Indicator | Normal | Mild | Moderate | Severe |
|-----------|--------|------|----------|--------|
| Shoulder height diff | < 12 | < 25 | < 40 | ≥ 40 |
| Kyphosis (hunchback) | < 8 | < 20 | < 35 | ≥ 35 |
| Head tilt | < 12 | < 22 | < 32 | ≥ 32 |
| Pelvic tilt | < 12 | < 25 | < 38 | ≥ 38 |
| Rounded shoulders | < 15 | < 30 | < 50 | ≥ 50 |

### Composite score formula

```
Score = shoulder×25% + kyphosis×25% + pelvis×20% + head×15% + rounded×15%
```

| Score | Grade |
|-------|-------|
| 90 – 100 | Excellent |
| 75 – 89 | Good |
| 60 – 74 | Fair |
| 45 – 59 | Poor |
| < 45 | Severe |

---

## Algorithm Details

### Keypoint depth projection (in-plane indicators)

The pinhole camera model converts pixel coordinate differences into real-world distances:

```
diff_mm(A, B) = |pixel_yA - pixel_yB| × z_ref / fy
```

`z_ref` — median depth sampled around the shoulder keypoints  
`fy` — vertical focal length from CameraInfo  
A **40-frame sliding median filter** suppresses per-frame jitter.

### Point cloud kyphosis algorithm

1. Define a torso ROI from shoulder and hip keypoints
2. Filter background: keep only points within 200 mm of the shoulder depth anchor
3. Divide the back region (normalized row range 0.50–0.85) into 20 horizontal layers; compute the depth median per layer
4. Average the nearest 20 % of layers → `back_z_min`
5. **Kyphosis = max(0, shoulder_depth − back_z_min − 60 mm)**

The **60 mm baseline** accounts for the natural anatomical depth offset between the shoulders and upper back in a healthy upright posture.

---

## FAQ

**Q: No output data during detection?**  
Ensure all three processes are running. The subject should stand facing the camera at 0.8–1.2 m with their full body visible in frame.

**Q: AI advice shows template text instead of personalized content?**  
Check that `DASHSCOPE_API_KEY` is set in `.env` and the board can reach the Alibaba Cloud API. Without a key the system uses local rule templates — this is expected behavior.

**Q: QR code scans but the report does not open?**  
Verify the phone and RDK X5 are on the same LAN. For remote access, set `CONFIG["ngrok_url"]` to a valid ngrok public URL.

**Q: Depth image shows excessive noise?**  
Verify camera calibration parameters are loaded correctly. Ensure adequate ambient lighting (avoid strong backlight) and use a plain-colored wall as background.

---

## Dependencies

### Board-side (`requirements.txt`)

```
mediapipe==0.10.14
opencv-python==4.10.0.84
dashscope==1.20.11
python-dotenv==1.0.1
jinja2==3.1.4
qrcode==7.4.2
```

### PC-side (GUI controller)

```
paramiko
qrcode
pillow
```

---

## License

MIT License
