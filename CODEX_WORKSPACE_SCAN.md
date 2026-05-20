# Codex Workspace Scan - sebot-t710-competition

Last scanned: 2026-05-20

Purpose: quick self-reference for future Codex work in this workspace. This is not a user manual and does not replace source-grounded inspection before edits.

## Top-Level Shape

This directory is a ROS/catkin competition workspace bundle, not a single git repository. The top level has three catkin workspaces plus a VS Code workspace file:

- `sebot_factory/`: main intelligent factory competition application. Contains task orchestration, object detection, arm control, picking/placing, checkout, and a Qt/RViz marking UI.
- `sebot_ros_kits/`: robot support stack. Contains chassis serial bridge, keyboard/joystick control, speech playback, camera/collection scripts, navigation stack, SLAM stack, lidar, URDF, robot pose EKF, MoveIt/Talon packages.
- `sebot_ros_stdr/`: STDR simulator stack and simulation launchers/navigation demos.
- `build/`: top-level generated CMake folder only. Each catkin workspace also has its own `build/` and `devel/`.
- `sebot-t710-competition.code-workspace`: VS Code workspace and C++ file associations.

There is no `.git` at the root in this checkout.

## Build Model

Each main subtree is already arranged as a catkin workspace:

- `sebot_factory/src/CMakeLists.txt`
- `sebot_ros_kits/src/CMakeLists.txt`
- `sebot_ros_stdr/src/CMakeLists.txt`

Expected Linux-side workflow is normally:

```bash
cd /root/workspace/sebot-t710-competition/sebot_ros_kits && catkin_make
source devel/setup.bash
cd /root/workspace/sebot-t710-competition/sebot_factory && catkin_make
source devel/setup.bash
```

For simulation, source `sebot_ros_stdr/devel/setup.bash` as needed.

Current generated binaries observed under `sebot_factory/devel/lib`:

- `sebot_factory/sebot_factory`
- `sebot_factory/sebot_yolo`
- `sebot_factory/sebot_summary`
- `sebot_factory/sebot_arm`
- `sebot_marking/sebot_marking`

Current generated binaries/libraries under `sebot_ros_kits/devel/lib` include `sebot_robot/sebot_controller`, `sebot_robot/sebot_joystick`, `sebot_navigation/multinavi`, `rplidar_ros/rplidarNode`, `move_base/move_base`, `map_server/map_server`, `amcl/amcl`, `gmapping/slam_gmapping`, and navigation libraries.

## Encoding Note

Many Chinese comments and launch comments render as mojibake in this Windows PowerShell scan. Treat source comments as likely GBK/legacy-encoded. Avoid broad re-encoding churn unless the user explicitly asks.

## Main Package: `sebot_factory`

Path: `sebot_factory/src/sebot_factory`

Role: full competition task node and standalone test nodes.

Important source files:

- `src/factory.cpp`: main autonomous factory state machine. Includes `confirm.cpp`, `picking.cpp`, and `summary.cpp` directly. Owns navigation, AMCL initial pose, audio prompts, task sequencing, and runtime parameter loading.
- `src/confirm.cpp`: demand/order confirmation task. Uses camera detection plus lidar/PID alignment. Publishes `/cmd_vel`.
- `src/picking.cpp`: intelligent picking/placing task. Uses lidar, RGB/depth camera, AI detection, Talon arm serial driver, and `/cmd_vel`.
- `src/summary.cpp`: checkout/order summary visualization logic.
- `unit/yolo.cpp`: standalone detection demo node `sebot_yolo`.
- `unit/pick.cpp`: standalone picking/placing test node `sebot_picking`, but its executable is currently commented out in CMake.
- `unit/order.cpp`: standalone order confirmation test node `sebot_confirm`, also currently commented out in CMake.
- `unit/pay.cpp`: builds as executable `sebot_summary`.
- `unit/arm.cpp`: builds as executable `sebot_arm`.

Important headers/resources:

- `include/tools.hpp`: common labels, `publishAudio`, `Pid`, `pidController`, `pdController`.
- `include/arm.hpp`: Talon mechanical arm serial protocol. Opens `/dev/talon` at 115200.
- `include/detection.hpp`: Paddle/PPNC + ONNX target detection wrapper. Loads model files from `res/model`.
- `include/camera.hpp`: V4L2 camera discovery helpers.
- `res/location.xml`: task/navigation point configuration used by factory logic and marking UI.
- `res/model/*`: detection model and post-processing assets. `Detection::buildNms()` untars/builds `nms.tar.so` at runtime.
- `res/image/*`: object/background images used by UI or summary.

Built executables from CMake:

- `sebot_factory`: `src/factory.cpp`
- `sebot_yolo`: `unit/yolo.cpp`
- `sebot_summary`: `unit/pay.cpp`
- `sebot_arm`: `unit/arm.cpp`

Commented out in CMake:

- `sebot_confirm`: `unit/order.cpp`
- `sebot_picking`: `unit/pick.cpp`

### `factory.cpp` Behavior Map

Core enums/classes:

- `FactoryStep`: `START -> INIT -> DELIVERY -> SUMMARY -> END`
- `StationNavi`: `TABLE`, `SERVING`, `COUNTER`
- `Table`: workbench goal, orders, ordered/putting flags, part placement count.
- `Location`: tables, serving point, origin point, counter point.
- `Navigation`: current AMCL pose, timeout, station target, arrival/overtime flags.

ROS interfaces:

- Publishes `/audio` as `std_msgs/String`.
- Publishes `/initialpose` for AMCL relocalization.
- Subscribes `/amcl_pose`.
- Uses `actionlib::SimpleActionClient<move_base_msgs::MoveBaseAction>` against `move_base`.

Runtime params under `/sebot_factory`:

- Startup/navigation: `delayStart`, `timeoutNavi`, `simulation`
- PID: `pidPoseKp/Ki/Kd`, `pidDisKp/Ki/Kd`, `pidLocalKp/Ki/Kd`, `pidClawXKp/Ki/Kd`, `pidClawYKp/Ki/Kd`
- Distances: `disClaw`, `disPick`, `disSearch`, `disOrder`
- Vision/debug: `debug`, `score`

High-level flow:

1. Start audio and initialization.
2. Relocalize against `location.origin` via `/initialpose` unless simulation is enabled.
3. Navigate to work table.
4. At table, run demand confirmation if needed, then place picked parts or go to serving area.
5. At serving area, run `Picking::pickupSomething()`.
6. Return to table for placement or proceed to counter.
7. At counter, enter summary UI loop.

### `confirm.cpp`

Role: confirm demand/order marker.

Important behavior:

- Subscribes `/scan` and publishes `/cmd_vel`.
- Uses lidar slices around front-left/front-right for pose/distance alignment.
- Uses AI labels from `tools.hpp`; order marker label is `LABEL_AI_ORDER`.
- Own state enum `ConfirmStep`: start, pose, part, end.
- Uses `Pid` instances for pose, distance, and local correction.

### `picking.cpp`

Role: pick or place physical parts with robot base plus Talon arm.

Important behavior:

- Subscribes `/scan`, publishes `/cmd_vel`.
- Uses `Talon` from `arm.hpp`.
- Uses RGB/depth camera selection through `setCamera()`.
- Uses `Detection` model wrapper and labels `nut`, `screw`, `pcb`, `block`, `tape`.
- Own state enum `PickStep`: start, pose, search, forward, aim, grab, local, end.
- Has public task-style entrypoints such as `pickupSomething()` and `putdownSomething()`.

Common object labels:

- `nut`
- `screw`
- `pcb`
- `block`
- `tape`
- `order`

## Launch Entrypoints in `sebot_factory`

Path: `sebot_factory/src/sebot_factory/launch`

- `sebot_factory.launch`: full real-robot factory flow. Starts audio, transform, `sebot_controller`, robot_pose_ekf, rplidar, map_server, AMCL, move_base, RViz, then `sebot_factory`.
- `sebot_picking.launch`: standalone picking test launch. Starts `sebot_controller`, rplidar, then `sebot_picking` params. Note: executable is commented out in current `CMakeLists.txt`, so generated binary may be absent unless CMake is changed.
- `sebot_yolo.launch`: standalone vision inference launch.
- `sebot_arm.launch`: standalone arm test launch.
- `sebot_ordering.launch`, `sebot_paying.launch`: standalone task launchers.

## Marking UI: `sebot_marking`

Path: `sebot_factory/src/sebot_marking`

Role: Qt5 + RViz GUI for marking/navigation points and map display.

Important files:

- `src/main.cpp`: Qt app entry.
- `src/qnode.cpp`, `include/qnode.hpp`: ROS thread. Publishes goals and receives `/multiGoal`.
- `src/qrviz.cpp`, `include/qrviz.hpp`: embeds RViz display/tool manager; supports set initial pose, set goal, multi-goal, display add/remove/rename/save/load.
- `src/slam.cpp`, `include/slam.h`: GUI orchestration for RViz, goal display widgets, process launch/kill behavior.
- `src/addtopics.cpp`, `include/addtopics.h`: topic/display selection UI.
- `resources/location.xml`, `resources/location.yaml`: stored point resources.

Launch:

- `sebot_factory/src/sebot_marking/launch/sebot_marking.launch`
  - Starts `map_server` with `$(find sebot_navigation)/map/map.yaml`.
  - Starts node `sebot_marking`.
  - Passes `pathXml=$(find sebot_factory)/res/location.xml`.

## Robot Support: `sebot_ros_kits/src/sebot_robot`

Role: robot base hardware bridge and manual control.

Important files:

- `src/controller.cpp`: node `sebot_controller`. Opens `/dev/robot` at 115200, subscribes `cmd_vel`, sends speed control over serial, publishes odom/IMU/ultrasonic topics if enabled.
- `include/uart.hpp`: lower-level serial protocol, frame definitions, odom/IMU/ultrasonic parsing, speed/heartbeat/reset/lidar commands.
- `src/joystick.cpp`: node `sebot_joystick`, reads `/dev/joystick`, publishes `cmd_vel`.
- `src/keyboards.cpp`: node `sebot_keyboard`, keyboard teleop, publishes `cmd_vel`.
- `src/transform.cpp`: node `sebot_transform`, publishes static robot transforms.
- `launch/sebot_odom.launch`, `sebot_keyboard.launch`, `sebot_joystick.launch`, `sebot_transform.launch`.

Key ROS interfaces:

- Input to chassis: `cmd_vel`
- Optional outputs: `odom`, `imu`, `msgUltraLF`, `msgUltraMF`, `msgUltraRF`, `msgUltraMB`
- TF: `odom -> base_footprint` when IMU fusion is not enabled.

## Speech: `sebot_ros_kits/src/sebot_speech`

Role: audio prompt playback.

Important file:

- `scripts/sebot_audio.py`: node `sebot_audio`, subscribes `/audio`, loads `/root/workspace/sebot-t710-competition/sebot_ros_kits/src/sebot_speech/res/audio/match/<name>.mp3`, falls back to `None.mp3`, plays with pygame.

Audio resources:

- `res/audio/match/*`: competition prompts, including `caterStart`, `confirm`, `gotoServing`, `pickNut`, `pickScrew`, `pickPcb`, `pickBlock`, `pickTape`, `pickOk`, `pickFailed`, `summary`, etc.
- `res/audio/sound/*`: generic system sounds.

Hard-coded Linux path in `sebot_audio.py` is important when running outside `/root/workspace/sebot-t710-competition`.

## Vision Scripts: `sebot_ros_kits/src/sebot_visions`

Role: Python scripts for joystick/data-collection style vision support.

Important files:

- `scripts/sebotCollection.py`: imports `JoyStick` and `Uart`, initializes UART on `/dev/robot`.
- `scripts/joyStick.py`: pygame joystick manager and handler.

This package has a minimal CMake/package wrapper and is separate from the C++ PPNC/ONNX inference in `sebot_factory/include/detection.hpp`.

## Navigation: `sebot_ros_kits/src/sebot_navigation`

Role: local copy/vendor bundle of ROS navigation plus project wrapper.

Project wrapper:

- `sebot_navigation/sebot_navigation`
  - `launch/sebot_navigation.launch`
  - `launch/sebot_multinavi.launch`
  - `param/amcl.xml`
  - `param/move_base.xml`
  - `param/*costmap*.yaml`
  - `param/planner_global_params.yaml`
  - `param/planner_local_params.yaml`
  - `map/map.yaml`, `map/map.pgm`
  - `rviz/navigation.rviz`
  - `src/multinavi.cpp`, `include/multinavi.hpp`

Bundled standard navigation packages include:

- `amcl`
- `base_local_planner`
- `carrot_planner`
- `clear_costmap_recovery`
- `costmap_2d`
- `dwa_local_planner`
- `fake_localization`
- `global_planner`
- `map_server`
- `move_base`
- `move_slow_and_clear`
- `nav_core`
- `navfn`
- `rotate_recovery`
- `voxel_grid`

When changing competition behavior, prefer editing the project wrapper/config first. Treat standard navigation packages as vendor code unless the user explicitly asks to patch them.

## SLAM: `sebot_ros_kits/src/sebot_slam`

Role: mapping stacks and project SLAM wrapper.

Project wrapper:

- `sebot_slam/sebot_slam`
  - `launch/sebot_gmapping.launch`
  - `launch/sebot_auto_gmapping.launch`
  - `launch/sebot_map_save.launch`
  - `param/gmapping.xml`
  - `param/hector.xml`
  - `param/auto_slam.xml`
  - `src/auto_slam.cpp`
  - `src/frontier_search.cpp`
  - `src/map2costmap.cpp`
  - `include/auto_slam.h`, `frontier_search.h`, `map2costmap.h`

Bundled stacks:

- `slam_gmapping/gmapping`
- `slam_hector/hector_mapping`
- `slam_hector/hector_map_server`
- `slam_hector/hector_map_tools`
- `slam_hector/hector_nav_msgs`
- `slam_hector/hector_marker_drawing`
- `slam_hector/hector_slam_launch`

## Driver/Model Packages: `sebot_ros_kits/src/sebot_driver`

Important packages:

- `rplidar_ros`: lidar driver; used by factory and picking launch files.
- `robot_pose_ekf`: odom/IMU fusion.
- `sebot_urdf`: robot model.
- `sebot_talon`: Talon-related driver package.
- `sebot_talon_moveit`: MoveIt config/launch package for Talon arm.

## STDR Simulation: `sebot_ros_stdr/src/sebot_stdr`

Role: STDR simulator, messages, robot/sensor models, map resources, and simulation navigation launchers.

Main packages:

- `stdr_simulator`
- `stdr_server`
- `stdr_gui`
- `stdr_robot`
- `stdr_msgs`
- `stdr_parser`
- `stdr_resources`
- `stdr_launchers`
- `stdr_navigation`
- `stdr_move_base`
- `stdr_amcl`
- `stdr_hector_mapping`
- `stdr_samples`

Useful launchers:

- `stdr_launchers/launch/server_no_map.launch`
- `stdr_launchers/launch/server_with_map.launch`
- `stdr_launchers/launch/server_with_map_and_gui.launch`
- `stdr_launchers/launch/server_with_map_and_gui_plus_robot.launch`
- `stdr_launchers/launch/sebot_map_and_gui.launch`
- `stdr_navigation/launch/*_nav.launch`
- `stdr_move_base/launch/stdr_move_base.launch`
- `stdr_amcl/launch/stdr_amcl.launch`
- `stdr_hector_mapping/launch/stdr_hector_mapping.launch`

## Runtime Data/Control Chain

Real robot full-flow launch path:

```text
sebot_factory.launch
  -> sebot_speech/sebot_audio.py
  -> sebot_robot/sebot_transform
  -> sebot_robot/sebot_controller
       /cmd_vel -> /dev/robot serial speed frames
       /dev/robot serial sensor frames -> odom/imu/ultrasonic topics
  -> robot_pose_ekf
  -> rplidar_ros/rplidar.launch -> /scan
  -> map_server -> /map
  -> sebot_navigation/param/amcl.xml -> /amcl_pose
  -> sebot_navigation/param/move_base.xml -> move_base action server
  -> rviz
  -> sebot_factory
       move_base goals + /initialpose + /audio + task modules
```

Picking path:

```text
factory.cpp or unit/pick.cpp
  -> Picking
     -> /scan lidar alignment
     -> /cmd_vel base motion
     -> camera via V4L2
     -> Detection PPNC/ONNX
     -> Talon arm over /dev/talon
```

Audio path:

```text
publishAudio(..., name)
  -> /audio std_msgs/String
  -> sebot_audio.py
  -> res/audio/match/name.mp3 or None.mp3
```

Navigation target path:

```text
location.xml
  -> factory.cpp XML parse
  -> move_base_msgs::MoveBaseGoal
  -> actionMoveBase.sendGoal(...)
  -> checkGoal(...) compares AMCL pose to target
```

## High-Value Search Anchors

Use these first for future work:

- PID implementation: `rg -n "class Pid|pidController|pdController" sebot_factory/src/sebot_factory/include/tools.hpp`
- PID params: `rg -n "pidPose|pidDis|pidLocal|pidClaw" sebot_factory/src/sebot_factory`
- Main task state machine: `rg -n "FactoryStep|caterHandle|taskTables|taskServing|naviToWorkStation" sebot_factory/src/sebot_factory/src/factory.cpp`
- Picking flow: `rg -n "PickStep|pickupSomething|putdownSomething|setCamera|robotCtrl" sebot_factory/src/sebot_factory/src/picking.cpp`
- Order confirmation: `rg -n "ConfirmStep|LABEL_AI_ORDER|orders|getConfirm" sebot_factory/src/sebot_factory/src/confirm.cpp`
- Arm serial protocol: `rg -n "USB_ADDR_|setActions|setJoint|setArmPose|/dev/talon" sebot_factory/src/sebot_factory/include/arm.hpp`
- Base serial protocol: `rg -n "USB_ADDR_|speedControl|transmitHart|/dev/robot|receiveThread" sebot_ros_kits/src/sebot_robot`
- Audio mapping: `rg -n "audioPath|Subscriber\\(\"/audio\"" sebot_ros_kits/src/sebot_speech/scripts/sebot_audio.py`
- Launch defaults: `rg -n "<param name=|<node pkg=|<include file=" sebot_factory/src/sebot_factory/launch sebot_factory/src/sebot_marking/launch`

## Cautions for Future Edits

- `sebot_factory/src/sebot_factory/src/factory.cpp` directly includes `.cpp` implementation files. Watch for duplicate symbols before enabling the commented standalone executables.
- `sebot_picking` and `sebot_confirm` launch files exist, but their executable targets are commented out in current `sebot_factory/CMakeLists.txt`.
- `sebot_audio.py` uses an absolute `/root/workspace/...` path; it is not portable to this Windows path without Linux-side matching path or a code change.
- `Detection::buildNms()` shells out to `tar` and `g++` at runtime. Model initialization can fail due to missing toolchain/files, not just C++ compile errors.
- `sebot_controller` uses relative topic name `cmd_vel`, which resolves to `/cmd_vel` when node namespace is root.
- Generated `build/` and `devel/` folders are present and should usually be ignored when editing source.
- Treat `sebot_ros_kits/src/sebot_navigation/*` and `sebot_ros_stdr/src/sebot_stdr/*` mostly as bundled upstream/vendor packages unless the requested change is explicitly in those stacks.
- Confirm current source before answering behavior questions; launch comments and existing docs can drift from active C++ logic.
