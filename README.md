# Autonomous Crack Detection Robot with LLM Reasoning

> **TTTC3413 — Robot Applications**  
> Universiti Kebangsaan Malaysia — Fakulti Teknologi & Sains Maklumat  
> **Topic:** Autonomous Crack Detection Robot for Industrial Infrastructure Inspection

---
## Overview

This project extends a YOLOv8-based autonomous crack detection robot into a fully intelligent inspection assistant by integrating Google Gemini, a Large Language Model (LLM), as a reasoning and decision-making layer.

Built on **TurtleBot3 Waffle Pi**, **ROS 1 Noetic**, and **Gazebo**, the system navigates autonomously within simulated industrial environments, detects pavement cracks in real time, classifies their severity using AI reasoning, and responds to natural language inspection queries, all without disrupting live robot operation.

The result is a shift from a passive detection tool to an **intelligent inspection assistant** capable of contextual understanding, structured reporting, and human-robot interaction.

---

## Features

- **Real-time crack detection** using YOLOv8 with bounding box visualisation and confidence scores
- **LLM-powered severity classification** — MINOR, MODERATE, or SEVERE — via the Google Gemini API
- **Natural language command interpretation** for robot motion control (forward, backward, left, right, stop)
- **Autonomous robot halting** upon crack detection for focused inspection
- **Structured inspection output** published over ROS topics for downstream consumption
- **Seamless ROS integration** — perception, reasoning, and control operate concurrently without pipeline bottlenecks

---


## Prerequisites

| Requirement | Version |
|---|---|
| Ubuntu | 20.04 LTS |
| ROS | Noetic Ninjemys |
| Python | 3.8+ |
| Gazebo | 11 |
| TurtleBot3 ROS Packages | Latest (Noetic) |
| YOLOv8 (`ultralytics`) | ≥ 8.0 |
| Google Generative AI SDK | `google-generativeai` |
| OpenCV | `opencv-python` |

---

## Workspace Structure

```
turtle_crack_ws2/
├── build/                        # CMake build artifacts (auto-generated)
├── devel/                        # Devel space — sourced for environment setup
│   ├── setup.bash
│   ├── setup.sh
│   └── ...
├── src/                          # All ROS packages live here
│   ├── crack_detection/          # Main package
│   │   ├── CMakeLists.txt
│   │   ├── package.xml
│   │   ├── launch/
│   │   │   └── crack_inspection.launch
│   │   ├── scripts/
│   │   │   ├── yolo_detector.py       # YOLOv8 crack detection node
│   │   │   └── llm_decision_node.py   # Gemini LLM reasoning node
│   │   ├── models/
│   │   │   └── best.pt                # Trained YOLOv8 weights
│   │   └── config/
│   │       └── params.yaml
│   └── CMakeLists.txt
└── .catkin_workspace              # Catkin workspace marker
```

---

## Installation

### 1. Clone the Repository

```bash
git clone <repository-url> ~/turtle_crack_ws2
cd ~/turtle_crack_ws2
```

### 2. Install ROS Dependencies

```bash
sudo apt update
sudo apt install ros-noetic-turtlebot3 \
                 ros-noetic-turtlebot3-simulations \
                 ros-noetic-turtlebot3-gazebo \
                 python3-rosdep python3-catkin-tools
```

### 3. Install Python Dependencies

```bash
pip install ultralytics \
            google-generativeai \
            opencv-python \
            numpy
```

### 4. Build the Workspace

```bash
cd ~/turtle_crack_ws2
catkin_make
source devel/setup.bash
```

Add to your shell profile to persist the environment:

```bash
echo "source ~/turtle_crack_ws2/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 5. Set TurtleBot3 Model

```bash
echo "export TURTLEBOT3_MODEL=waffle_pi" >> ~/.bashrc
source ~/.bashrc
```

---

## Configuration

### Gemini API Key

Obtain a free API key from [Google AI Studio](https://aistudio.google.com/app/apikey) and export it as an environment variable:

```bash
export GEMINI_API_KEY="your_api_key_here"
```

Or add it permanently to your shell profile:

```bash
echo 'export GEMINI_API_KEY="your_api_key_here"' >> ~/.bashrc
source ~/.bashrc
```

### YOLOv8 Model Weights

Place your trained YOLOv8 weights (`best.pt`) in the `src/crack_detection/models/` directory. Update the model path in `yolo_detector.py` if necessary:

```python
model = YOLO("/path/to/turtle_crack_ws2/src/crack_detection/models/best.pt")
```

---

## Usage

### Launch the Full Inspection System

Open separate terminals for each component:

**Terminal 1 — Gazebo Simulation**

```bash
source ~/turtle_crack_ws2/devel/setup.bash
roslaunch turtlebot3_gazebo turtlebot3_world.launch
```

**Terminal 2 — YOLO Crack Detector**

```bash
source ~/turtle_crack_ws2/devel/setup.bash
rosrun crack_detection yolo_detector.py
```

**Terminal 3 — Gemini LLM Decision Node**

```bash
source ~/turtle_crack_ws2/devel/setup.bash
rosrun crack_detection llm_decision_node.py
```

**Terminal 4 — Image Viewer (optional)**

```bash
rosrun rqt_image_view rqt_image_view /camera/yolo_image
```

### Alternatively, Use the Launch File

```bash
source ~/turtle_crack_ws2/devel/setup.bash
roslaunch crack_detection crack_inspection.launch
```

### Controlling the Robot

Issue natural language movement commands through the LLM interface:

| Command | Action |
|---|---|
| `forward` | Move robot forward |
| `backward` | Move robot backward |
| `left` | Turn robot left |
| `right` | Turn robot right |
| `stop` | Halt robot immediately |

The robot will **automatically stop** when a crack is detected, regardless of the active command.

---

## ROS Topics

| Topic | Type | Description |
|---|---|---|
| `/camera/rgb/image_raw` | `sensor_msgs/Image` | Raw camera feed from TurtleBot3 |
| `/camera/yolo_image` | `sensor_msgs/Image` | Annotated image with bounding boxes |
| `/crack_detection/crack_image` | `sensor_msgs/Image` | Cropped image of detected crack |
| `/crack_detection/severity` | `std_msgs/String` | Severity classification output |
| `/cmd_vel` | `geometry_msgs/Twist` | Robot velocity commands |

---

## Module Descriptions

### `yolo_detector.py`

The primary perception node. Subscribes to the live camera feed (`/camera/rgb/image_raw`) and runs YOLOv8 inference on each frame. Detected cracks are annotated with bounding boxes and confidence scores, then published to `/camera/yolo_image` for visualisation in `rqt_image_view`. When a crack is detected, the cropped region is extracted and published to `/crack_detection/crack_image` for LLM analysis.

### `llm_decision_node.py`

The intelligent reasoning node. Subscribes to `/crack_detection/crack_image` and forwards each detected crack image to the Google Gemini API via the `google-generativeai` library. Gemini analyses the visual content and returns a semantic severity classification (MINOR, MODERATE, or SEVERE). The node then:

1. Publishes the severity result to `/crack_detection/severity`
2. Issues a stop command to `/cmd_vel` to halt the robot at the inspection site
3. Logs the detection event with timestamp and location coordinates

## Authors

| Name | Matric Number |
|---|---|
| Nancy Anne | A202459 |
| Kavi Priya A/P Balasupramaniam | A202660 |

**Lecturer:** Dr. Anahita Ghazvini  
**Course:** TTTC3413 — Robot Applications  
**Institution:** Universiti Kebangsaan Malaysia (UKM)

---

## References

- Ai, D., Jiang, G., Kei, L. S. & Li, C. (2018). Automatic pixel-level pavement crack detection using information of multi-scale neighborhoods. *IEEE Access*, 6, 24452–24463.
- Cuevas, D. & Manrique, R. (2025). Evaluating LLM-Based Autonomous Systems. *ICAI 2025*.
- Ibrahimi, I. (2024). Integrating Real-Time Object Detection with LiDAR Data for Enhanced Robotic Autonomous Navigation. *Politecnico di Torino*.
- Kim, Y., Kim, D., Choi, J., Park, J., Oh, N. & Park, D. (2024). A survey on integration of large language models with intelligent robots. *Intelligent Service Robotics*, 17(5), 1091–1107.
- Lee, D., Nie, G.-Y. & Han, K. (2023). Vision-based inspection of prefabricated components using camera poses. *Journal of Building Engineering*, 64, 105710.
- Marian, M. et al. (2020). A ROS-based control application for a robotic platform using the Gazebo 3D simulator. *ICCC 2020*.
- Sun, C., Huang, S. & Pompili, D. (2025). LLM-Based Multi-Agent Decision-Making. *IEEE Robotics and Automation Letters*.
- Team, G. et al. (2023). Gemini: a family of highly capable multimodal models. *arXiv:2312.11805*.

---

<div align="center">

**Fakulti Teknologi & Sains Maklumat — Universiti Kebangsaan Malaysia**

*Built with ROS Noetic · YOLOv8 · Google Gemini · Gazebo · TurtleBot3*

</div>
