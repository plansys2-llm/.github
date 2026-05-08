# plansys2-llm

LLM-assisted replanning on a mobile robot, demoed in a service-robot scenario: a Kobuki picks up a misplaced book in a bookstore and returns it to the right shelf.

This organization owns two of the packages used by the demo. The full installation flow — including the upstream stacks the demo composes — is documented below.

---

## Repositories

| Repository | What it is |
|---|---|
| [`plansys2_llm_solver`](https://github.com/plansys2-llm/plansys2_llm_solver) | LLM-assisted replanner for PlanSys2. Invoked at runtime when execution fails or perception contradicts the world model and returns the predicate deltas needed to recover. |
| [`plansys2_llm_examples`](https://github.com/plansys2-llm/plansys2_llm_examples) | The `plan_bookstore` demo: PDDL domain, behavior trees, launch files, maps, and book models — entry point of the project. |

---

## Stack overview

- **Robot:** Kobuki (simulated)
- **Simulator:** Gazebo Sim with the AWS RoboMaker bookstore world
- **Navigation:** EasyNav (AMCL localizer + costmap planner/controller)
- **Perception:** YOLO via `yolo_ros`
- **PDDL planning:** PlanSys2 with POPF as the planner backend
- **LLM replanner:** `plansys2_llm_solver` (this project) — runs alongside POPF and is consulted at execution time
- **Behavior trees:** PlanSys2 BT actions (`move`, `pick_book`, `place_book`)

---

## Installation

Tested on **Ubuntu 24.04** with **ROS 2 Jazzy** and **ROS 2 Rolling**. The instructions below use `${ROS_DISTRO}` so the same recipe works on either; set it once and the rest of the steps follow.

```bash
export ROS_DISTRO=jazzy   # or: rolling
source /opt/ros/${ROS_DISTRO}/setup.bash
```

### 1. System dependencies

```bash
sudo apt install -y \
  python3-colcon-common-extensions python3-rosdep python3-vcstool \
  ros-${ROS_DISTRO}-ros-gz-sim ros-${ROS_DISTRO}-ros-gz-bridge \
  nlohmann-json3-dev \
  libusb-1.0-0-dev libftdi1-dev libuvc-dev

# ultralytics and lap are not packaged in apt; pip is the only option.
# --break-system-packages is required by PEP 668 on Ubuntu 24.04.
# The numpy<2 pin prevents pip from upgrading numpy and breaking cv_bridge.
pip install --break-system-packages "numpy<2" ultralytics lap
```

> `libusb-1.0-0-dev`, `libftdi1-dev` and `libuvc-dev` are required by `kobuki_core` and `astra_camera`; without them the first `colcon build` fails.

> Do **not** install `opencv-python` from pip — it shadows the system `cv2` that `cv_bridge` was built against and causes a SIGSEGV in `yolo_node`.

### 2. Create the workspace

```bash
mkdir -p ~/TFG/src/{llm,navigation,perception,planning,robot/ThirdParty}
cd ~/TFG/src
```

### 3. Clone the repositories

```bash
# LLM
git clone -b ${ROS_DISTRO} https://github.com/mgonzs13/llama_ros.git llm/llama_ros

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

----

**EasyNav — pick one of the two paths.** EasyNav can be installed as a Debian package or built from source. **Choose one route only.** Mixing the apt deb and a source overlay invites ABI drift between releases.

*Option A — apt deb (recommended).* Pulls the metapackage and all bundled plugins from the ROS 2 binary repository. Faster, no rebuild on EasyNav changes.

```bash
sudo apt install -y ros-${ROS_DISTRO}-easynav

# AMCLLocalizer is not always shipped as a deb — clone just easynav_plugins:
git clone -b ${ROS_DISTRO} https://github.com/EasyNavigation/easynav_plugins.git \
  navigation/easynav_plugins
```

*Option B — from source.* Builds the whole EasyNav stack from source. Pick this if you need to track upstream `main`/`devel` or modify EasyNav itself.

```bash
git clone -b ${ROS_DISTRO} https://github.com/EasyNavigation/EasyNavigation.git \
  navigation/EasyNavigation
git clone -b ${ROS_DISTRO} https://github.com/EasyNavigation/easynav_plugins.git \
  navigation/easynav_plugins
```

> If you take Option B, do **not** also install `ros-${ROS_DISTRO}-easynav` from apt.

### 4. Build

```bash
cd ~/TFG
sudo rosdep init || true   # safe to skip if already initialised
rosdep update
rosdep install --from-paths src --ignore-src -y -r --rosdistro ${ROS_DISTRO}

# CPU only:
colcon build --symlink-install --cmake-args -DBUILD_TESTING=OFF

# Or, with NVIDIA GPU acceleration for llama_ros (requires the CUDA toolkit
# matching your driver — verify with `nvcc --version`):
colcon build --symlink-install --cmake-args -DBUILD_TESTING=OFF -DGGML_CUDA=ON
```

> If the build runs out of RAM or the terminal crashes (common when `llama_cpp_vendor` and `plansys2` compile in parallel), retry with a single worker: `colcon build --symlink-install --parallel-workers 1`. You can also limit it to the failing package with `--packages-select <pkg>`, or drop a `COLCON_IGNORE` file inside a package to skip it.

---

## Run

In the first terminal, source the workspace and bring up the simulator + navigation + perception + PlanSys2 stack:

```bash
source /opt/ros/${ROS_DISTRO}/setup.bash
source ~/TFG/install/setup.bash

ros2 launch plan_bookstore bookstore_kobuki_launch.py \
  perception_mode:=sim displaced_book:=red_book
```

Launch arguments:

- `displaced_book` — `red_book`, `green_book`, `yellow_book`, `blue_book`
- `perception_mode` — `sim` (synthetic detections, no extra setup) or `real` (YOLO via `yolo_ros`; the model weights are managed by `yolo_ros` itself)
- `gui` — `true` (default) launches Gazebo with its GUI; `false` runs headless

The first launch with `perception_mode:=real` may take longer because `yolo_ros` and `llama_ros` fetch their model weights on demand.

Then, in a second terminal (with the same two `source` lines), start the reception controller — this is the node that drives the demo and consults the LLM solver when replanning is needed:

```bash
ros2 run plan_bookstore reception_controller_node \
  --ros-args \
  --params-file src/planning/plansys2_llm_examples/params/planner_param.yaml \
  -p displaced_book:=red_book
```

Pick whichever `planner_param*.yaml` you want to test — `plansys2_llm_examples/params/` ships several profiles (e.g. `planner_param.yaml`, `planner_param_with_args.yaml`); swap the path to switch the planner/solver configuration without rebuilding.

---

## Optional tweaks

These changes live in third-party repos and cannot be committed here. Apply them after step 3 if you want a smoother bookstore demo.

### Kobuki LiDAR — 360° field of view

The default LiDAR is a 180° frontal sweep; AMCL and the costmap behave better with the full 360°.

Edit `src/robot/ThirdParty/kobuki_ros/kobuki_description/urdf/kobuki_gazebo.urdf.xacro`, lidar `<horizontal>` block:

```xml
<min_angle>-3.1416</min_angle>   <!-- was -1.5708 -->
<max_angle>3.1416</max_angle>    <!-- was  1.5708 -->
```

Rebuild:

```bash
colcon build --packages-select kobuki_description --symlink-install
```

### Phi-4 — tune for CPU-only deployments

Defaults are single-threaded with a tiny batch. For 4-core CPUs (e.g. Raspberry Pi 5 16 GB), edit `src/llm/llama_ros/llama_bringup/models/Phi-4.yaml`:

```yaml
context:
  n_ctx: 4096      # was 2048 — longer context for replanning prompts
  n_batch: 256     # was 8    — fewer forward passes per token
  n_predict: 1024  # was 2048 — caps response length to keep latency bounded
cpu:
  n_threads: 4     # was 1    — match the CPU's physical cores
```

No rebuild needed.

---

## Real robot (optional)

The bookstore demo runs in Gazebo, so the steps above are enough. If you also want to drive a **physical Kobuki**, you need udev rules so the kernel exposes `/dev/kobuki`, `/dev/rplidar`, etc. with the right permissions:

```bash
cd ~/TFG
sudo cp src/robot/ThirdParty/ros_astra_camera/astra_camera/scripts/56-orbbec-usb.rules /etc/udev/rules.d/
sudo cp src/robot/ThirdParty/rplidar_ros/scripts/rplidar.rules /etc/udev/rules.d/
sudo cp src/robot/ThirdParty/kobuki_ros/60-kobuki.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Then launch the real-robot stack (sensors are off by default):

```bash
ros2 launch kobuki kobuki.launch.py            # base only
ros2 launch kobuki kobuki.launch.py lidar_a2:=true   # or lidar_s2 / xtion / astra
```

Refer to the upstream guide for hardware-specific quirks: <https://github.com/IntelligentRoboticsLabs/kobuki/tree/jazzy>.

---

## Troubleshooting

If something specific to the Kobuki / EasyNav / PlanSys2 stacks fails and the notes above don't help, check the upstream repos — they are the source of truth and may have moved on since this guide was written:

- Kobuki meta-repo (drivers, worlds, launchers): <https://github.com/IntelligentRoboticsLabs/kobuki/tree/jazzy>
- EasyNavigation core: <https://github.com/EasyNavigation/EasyNavigation>
- PlanSys2: <https://github.com/PlanSys2/ros2_planning_system>
- llama_ros: <https://github.com/mgonzs13/llama_ros>
- yolo_ros: <https://github.com/mgonzs13/yolo_ros>

---

## Project context

Bachelor's Final Project (TFG) at Universidad Rey Juan Carlos (URJC).

---

## License

The two repositories owned by this organization (`plansys2_llm_solver`, `plansys2_llm_examples`) are released under Apache 2.0. Upstream stacks pulled in during installation keep their own licenses — see each repository's `LICENSE` file.
