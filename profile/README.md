# plansys2-llm

LLM-assisted replanning on a mobile robot, demoed in a service-robot scenario: a Kobuki picks up a misplaced book in a bookstore and returns it to the right shelf.

This organization hosts the two repositories that make up the project. The full installation and run instructions live here.

---

## Repositories

| Repository | What it is |
|---|---|
| [`plansys2_llm_solver`](https://github.com/plansys2-llm/plansys2_llm_solver) | LLM-assisted replanner for PlanSys2. Invoked at runtime when execution fails or perception contradicts the world model; returns the predicate deltas needed to recover. Default plugin uses local llama.cpp via `llama_ros`; other LLMs (ChatGPT, Claude, …) can be plugged in. |
| [`plansys2_llm_examples`](https://github.com/plansys2-llm/plansys2_llm_examples) | The `plan_bookstore` demo: PDDL domain, behavior trees, launch files, maps, and book models. Entry point of the project. |

---

## Stack overview

- **Robot:** Kobuki (simulated)
- **Simulator:** Gazebo Sim with the AWS RoboMaker bookstore world
- **Navigation:** EasyNav (AMCL localizer + costmap planner/controller)
- **Perception:** YOLO via `yolo_ros`
- **PDDL planning:** PlanSys2 with POPF as the planner backend
- **LLM replanner:** `plansys2_llm_solver` (this project) — runs alongside POPF and is consulted at execution time. Default plugin: local llama.cpp via `llama_ros`; pluggable for other LLMs.
- **Behavior trees:** PlanSys2 BT actions (`move`, `pick_book`, `place_book`)

---

## Installation

Tested on **Ubuntu 24.04 + ROS 2 Jazzy**. Replace `${ROS_DISTRO}` below with your distro (`jazzy` or `rolling`).

### 1. Install dependencies

```bash
export ROS_DISTRO=jazzy
source /opt/ros/${ROS_DISTRO}/setup.bash

sudo apt install -y \
  python3-colcon-common-extensions python3-rosdep python3-vcstool \
  ros-${ROS_DISTRO}-ros-gz-sim ros-${ROS_DISTRO}-ros-gz-bridge \
  ros-${ROS_DISTRO}-easynav nlohmann-json3-dev

pip install --break-system-packages "numpy<2" ultralytics lap
```

> Do **not** install `opencv-python` from pip — it shadows the system `cv2` that `cv_bridge` was built against and causes a SIGSEGV in `yolo_node`.

### 2. Create the workspace

```bash
mkdir -p ~/TFG/src/{llm,navigation,perception,planning,robot/ThirdParty}
cd ~/TFG/src
```

### 3. Clone the repositories

Main repos:

```bash
# LLM
git clone -b ${ROS_DISTRO} https://github.com/mgonzs13/llama_ros.git llm/llama_ros

# Navigation
git clone -b ${ROS_DISTRO} https://github.com/EasyNavigation/EasyNavigation.git navigation/EasyNavigation
git clone -b ${ROS_DISTRO} https://github.com/EasyNavigation/easynav_plugins.git navigation/easynav_plugins

# Perception
git clone https://github.com/mgonzs13/yolo_ros.git perception/yolo_ros

# Planning
git clone -b ${ROS_DISTRO}-devel https://github.com/PlanSys2/ros2_planning_system.git planning/ros2_planning_system
git clone https://github.com/plansys2-llm/plansys2_llm_solver.git planning/plansys2_llm_solver
git clone https://github.com/plansys2-llm/plansys2_llm_examples.git planning/plansys2_llm_examples

# Robot (meta-package — declares its own ThirdParty deps in thirdparty.repos)
git clone -b ${ROS_DISTRO} https://github.com/IntelligentRoboticsLabs/kobuki.git robot/kobuki
```

Pull the third-party dependencies declared inside the meta repos:

```bash
vcs import robot < robot/kobuki/thirdparty.repos
vcs import planning/ros2_planning_system < planning/ros2_planning_system/dependency_repos.repos
```

### 4. Build

```bash
cd ~/TFG
rosdep install --from-paths src --ignore-src -y -r --rosdistro ${ROS_DISTRO}

# CPU only:
colcon build --symlink-install --cmake-args -DBUILD_TESTING=OFF

# Or, with NVIDIA GPU acceleration for llama_ros (requires CUDA toolkit):
colcon build --symlink-install --cmake-args -DBUILD_TESTING=OFF -DGGML_CUDA=ON
```

---

## Run

```bash
source /opt/ros/${ROS_DISTRO}/setup.bash
source ~/TFG/install/setup.bash

ros2 launch plan_bookstore bookstore_kobuki_launch.py \
  perception_mode:=real displaced_book:=red_book
```

Launch arguments:

- `displaced_book` — `red_book`, `green_book`, `yellow_book`, `blue_book`
- `perception_mode` — `real` (YOLO) or `sim` (synthetic detections)
- `gui` — `true` / `false`

---

## Project context

Bachelor's Final Project (TFG) at Universidad Rey Juan Carlos (URJC).

## License

Apache 2.0 — see each repository's `LICENSE` file.
