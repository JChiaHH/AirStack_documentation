# Operation

> File paths in this document are relative to the **AirStack repo root** (e.g. `~/AirStack`), not this documentation repo.

[← Docs index](README.md) · [Getting Started](getting-started.md) · [Architecture](architecture.md) · [Configuration](configuration.md)

## Open the Visualizer in Foxglove

AirStack uses **Foxglove Studio** as its default visualizer (not RViz in this version). After `airstack up`, a Foxglove window opens but starts with no layout and no live connection — load the layout and connect:

1. Click the **top-left icon** (the Foxglove menu).
2. Click **"Layouts"**.
3. Click **"+ Add"**.
4. Click **"Import Personal Layout"**.
5. Select the layout file **`airstack_layout_num_robots_1.json`** (matches `NUM_ROBOTS=1`; for N robots, pick the `_num_robots_<N>` file). It is generated inside the GCS container at `/root/` on every startup.
6. Open the selected file in the dialog.
7. Click the **top-left icon** again.
8. Click **"Open connection"**.
9. In the **Foxglove WebSocket** row, confirm the address bar shows `ws://localhost:8765` and click **Open**.
   - Use `ws://localhost:8765` when Foxglove runs **inside** the GCS container.
   - Use `ws://localhost:8766` when Foxglove runs on the **host**.

You should now see the drone and live data in the visualizer.

**To refresh Foxglove** (e.g. after restarting the sim): re-open the connection (steps 7–9), or hard-reload the window with **`Ctrl+R`** (your imported layout is preserved).

---

## Reset Isaac Sim Without Restarting AirStack

If only the simulator misbehaves (e.g. PX4 died), you don't need to tear down the whole stack. From lightest to heaviest:

**Option 1 — Stop → Play in the Isaac Sim GUI (lightest).**
In Pegasus, PX4's lifecycle is tied to the simulation timeline: pressing **Stop** kills the stale PX4 and closes the MAVLink link; pressing **Play** re-initializes the link and **auto-relaunches a fresh PX4**. Try this first. (If the backend is already wedged mid-crash, this may not recover — escalate.)

**Option 2 — Restart just the Isaac Sim container.**

```bash
docker restart isaac-sim
```

Because `AUTOLAUNCH="true"`, this re-runs the launch script and spawns a fresh PX4. MAVROS on the robot reconnects automatically. The robot and GCS containers keep running.

**Option 3 — Full clean restart (most reliable).**

```bash
cd AirStack
airstack down
airstack up
```

> **Note:** There is no first-class "restart PX4 only" command, and the Pegasus MAVLink backend has no auto-reconnect logic. Relaunching only the PX4 binary by hand will **not** restore telemetry — use one of the options above.

After restarting, verify before commanding flight:

```bash
# PX4 alive (should show a running px4, NOT "<defunct>")
docker exec isaac-sim bash -lc 'pgrep -af px4'

# Odometry flowing (want a steady Hz, not "no new messages")
docker exec airstack-robot-desktop-1 bash -lc \
  'source /root/AirStack/robot/ros_ws/install/setup.bash && \
   timeout 5 ros2 topic hz /robot_1/odometry_conversion/odometry'
```

---

## ROS 2 Topics

Robot-scoped topics are namespaced under `/{robot_name}` (e.g. `/robot_1`); GCS topics live under `/gcs`. In a multi-robot deployment each robot runs on its own `ROS_DOMAIN_ID`, and the GCS bridges per-robot topics across domains.

> **Frames:** Each robot publishes its autonomy data in its own local `map` frame (origin = the drone's boot/takeoff position). The GCS visualizer georeferences these into a single shared global ENU `map` frame using each robot's first GPS fix as a boot offset.

### Viewing topics across DDS domains

> **If a topic you expect is missing from `ros2 topic list`, you are almost always on the wrong DDS domain — the topic exists, your terminal just can't see it.** This is the single most common point of confusion in AirStack. Read this section before assuming a topic is broken or absent.

#### What is `ROS_DOMAIN_ID`?

ROS 2 nodes only discover and talk to other nodes that share the same **`ROS_DOMAIN_ID`** — an integer (0–101) that partitions the network into isolated groups. Nodes on domain 1 cannot see topics on domain 0, and vice versa, even on the same machine. It's like a channel number: everyone must be tuned to the same channel to hear each other. If `ROS_DOMAIN_ID` is unset, it defaults to **0**.

AirStack uses this deliberately to keep robots isolated:

| Who | `ROS_DOMAIN_ID` | Sees... |
|---|---|---|
| Your host terminal (default) | **0** | Only the GCS domain + topics bridged to it |
| GCS container | **0** | GCS domain |
| `robot_1` container | **1** | All of robot_1's topics (full `mavros/*`, sensors, etc.) |
| `robot_2` container | **2** | All of robot_2's topics |
| robot N container | **N** | All of robot N's topics |

A **DDS router** copies a curated **allowlist** of each robot's topics onto domain 0 so the GCS (and your host) can see them. Everything *not* on that allowlist stays private to the robot's domain.

#### Where each topic lives

| Topic group | Domain | Visible from host by default? |
|---|---|---|
| `/gcs/*`, `/clock`, `/tf_static` | 0 (GCS) | ✅ Yes |
| **Bridged** robot topics: `…/odometry_conversion/odometry`, `…/mavros/global_position/global`, `…/sensors/*`, `…/tasks/*`, `…/global_plan`, `…/vdb_mapping/*`, `…/trajectory_controller/trajectory_vis` | copied 0 ← N | ✅ Yes |
| **Robot-only** topics: most `…/mavros/*` — `local_position/odom`, `local_position/pose`, `imu/data`, `estimator_status`, `altitude`, `state`, `extended_state`, `battery` | N (robot) | ❌ No — must be on domain N |

The exact allowlist is defined in `robot/ros_ws/src/autonomy_bringup/onboard_all/config/dds_router.yaml`. So when you don't see `…/mavros/local_position/odom` on your host, it's because that topic is **only on domain 1** and isn't in the allowlist — not because it's missing.

#### How to set a terminal to a different domain

`ROS_DOMAIN_ID` is just an environment variable. You can set it three ways:

```bash
# 1) For a SINGLE command (prefix it — does not affect your shell afterward):
ROS_DOMAIN_ID=1 ros2 topic list
ROS_DOMAIN_ID=1 ros2 topic echo /robot_1/interface/mavros/local_position/odom

# 2) For the WHOLE terminal session (every later ros2 command uses domain 1):
export ROS_DOMAIN_ID=1
ros2 topic list                 # now shows robot_1's full topic set
ros2 topic echo /robot_1/interface/mavros/local_position/pose
# ...switch back when done:
export ROS_DOMAIN_ID=0          # (or just open a new terminal)

# 3) Check what domain your terminal is currently on:
echo "ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-0 (unset = default 0)}"
```

> **Important:** This only works if your **host has a ROS 2 environment installed and sourced** (e.g. `source /opt/ros/jazzy/setup.bash`). If `ros2` isn't found on your host, or you're unsure it matches the container's ROS version, use the container instead (option B below) — it always has the correct environment.

#### The three ways to view a robot-only topic

```bash
# (A) Use the BRIDGED equivalent from the host as-is (no domain change needed).
#     odometry_conversion/odometry carries the same EKF data as local_position/odom:
ros2 topic echo /robot_1/odometry_conversion/odometry

# (B) Run ros2 INSIDE the robot container (already on domain 1, env guaranteed):
docker exec airstack-robot-desktop-1 bash -lc \
  'source /root/AirStack/robot/ros_ws/install/setup.bash; \
   ros2 topic echo /robot_1/interface/mavros/local_position/odom'

# (C) Switch your HOST terminal onto the robot's domain (needs host ROS 2 sourced):
export ROS_DOMAIN_ID=1
ros2 topic echo /robot_1/interface/mavros/local_position/odom
export ROS_DOMAIN_ID=0     # reset afterward to get GCS/bridged topics back
```

Use **(A)** when the bridged topic already gives you what you need (it usually does), **(B)** as the most reliable way to inspect any robot-only topic, and **(C)** when you want to browse the robot's full topic list from your own terminal.

> **Tip — multi-robot:** to inspect `robot_2`, use `ROS_DOMAIN_ID=2` (or `docker exec airstack-robot-desktop-2 ...`). Each robot N is on domain N.

### 1. Pose & State Estimation (Onboard EKF)

The estimation chain is: **PX4 onboard EKF2 → MAVROS → `odometry_conversion` → autonomy stack.** PX4's EKF2 fuses IMU/GPS/baro onboard and exposes its estimate over MAVLink; the MAVROS `local_position` plugin converts it to ENU and publishes it (already in the `map`→`base_link` frame) on `/{robot}/interface/mavros/local_position/odom`. The `odometry_conversion` node subscribes to that topic directly and re-publishes it as **`/{robot}/odometry_conversion/odometry`** — the **canonical AirStack odometry** that every autonomy module (planners, controllers, takeoff/landing, safety monitor) consumes via the remapped `odometry` topic.

> **Raw mavros odom vs. converted odom — what actually differs.** Both topics carry the *same* PX4 EKF estimate (identical position, orientation, velocity, and timestamp). `odometry_conversion` is a thin republisher, not a separate estimator. In this MAVROS-based deployment the only real differences are:
> - **QoS:** the mavros topic is `BEST_EFFORT`; the converted topic is `RELIABLE` (so the autonomy stack receives every sample). This is the node's main purpose.
> - **TF:** `odometry_conversion` also broadcasts the dynamic transforms `map→base_link` and `map→base_link_stabilized`; mavros does not publish TF.
> - The frame IDs are already `map`/`base_link` on the mavros topic (set in `px4_config.yaml`), so the node's frame "overwrite" is effectively a no-op here.
>
> **Covariance is unpopulated.** The pose/twist covariance is **all zeros on both topics** — PX4's EKF2 does not fill the covariance fields of the MAVLink ODOMETRY message in this setup. Neither topic provides EKF uncertainty; to get real covariance you must enable it upstream (PX4 / MAVLink odometry).
>
> **Note (legacy uXRCE-DDS path):** an alternate interface (`px4_interface.launch.xml`) exists in the repo that adds a `/{robot}/interface/odometry` hop, but it is **not** what runs by default. The default stack uses MAVROS via `interface_bringup/launch/interface.launch.py`, and there is no `/{robot}/interface/odometry` topic in a default deployment.

> **Visibility column:** **Host** = bridged to the GCS/host domain (`ROS_DOMAIN_ID=0`) by the DDS router, so it appears in `ros2 topic list` on your laptop. **Robot only** = lives on the robot's domain (`ROS_DOMAIN_ID=N`) and is **not** bridged — it exists and publishes, but you must be on the robot domain to see it (see [Viewing topics across DDS domains](#viewing-topics-across-dds-domains)).

| Topic | Type | Purpose | Visibility | Notes |
|---|---|---|---|---|
| `/{robot}/interface/mavros/local_position/odom` | `nav_msgs/Odometry` | PX4 onboard EKF pose + twist estimate as exposed by MAVROS | **Robot only** | Raw MAVROS output; BEST_EFFORT QoS. Frame `map`→`base_link`. Upstream source of AirStack odometry. |
| `/{robot}/interface/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | PX4 EKF pose only (no twist) | **Robot only** | Pose-only view of the same EKF estimate. |
| `/{robot}/odometry_conversion/odometry` | `nav_msgs/Odometry` | **Canonical AirStack odometry** used by the whole autonomy stack | **Host** | Frame `map` → child `base_link`; RELIABLE QoS; also broadcast to TF. Modules subscribe to this (remapped to `odometry`). Source for the GCS pose arrow. Numerically identical to the raw mavros odom. |
| `/{robot}/interface/mavros/imu/data` | `sensor_msgs/Imu` | Orientation + angular velocity + linear acceleration from the flight-controller IMU | **Robot only** | Feeds PX4 EKF; available to perception/VIO. |
| `/{robot}/interface/mavros/estimator_status` | `mavros_msgs/EstimatorStatus` | **PX4 EKF health/status flags** (attitude, velocity, position validity, etc.) | **Robot only** | Use to check estimator health. |
| `/{robot}/interface/mavros/global_position/global` | `sensor_msgs/NavSatFix` | GPS global position (lat/lon/alt) | **Host** | The GCS uses the first valid fix as the robot's ENU "boot" origin; `action_relay` gates non-takeoff tasks on its altitude. |
| `/{robot}/interface/mavros/altitude` | `mavros_msgs/Altitude` | PX4 altitude estimates (AMSL, relative, terrain) | **Robot only** | |
| `/{robot}/interface/mavros/state` | `mavros_msgs/State` | FCU connection/armed/mode (e.g. OFFBOARD) state | **Robot only** | |
| `/{robot}/interface/mavros/extended_state` | `mavros_msgs/ExtendedState` | Landed state (ON_GROUND / IN_AIR) and VTOL state | **Robot only** | Takeoff/landing task uses `landed_state` to confirm airborne/landed. |
| `/{robot}/interface/mavros/battery` | `sensor_msgs/BatteryState` | Battery voltage / percentage | **Robot only** | |
| `/{robot}/behavior/drone_safety_monitor/state_estimate_timed_out` | `std_msgs/Bool` | Watchdog: true if odometry stopped arriving within the timeout | **Robot only** | Published at 1 Hz by `drone_safety_monitor`, which watches `odometry_conversion/odometry`. On timeout it auto-pauses the trajectory controller; the takeoff task rejects new goals while true. |

> Most `mavros/*` topics are **robot-domain-only** by default — only `odometry_conversion/odometry` and `mavros/global_position/global` from this group are bridged to the host. If you don't see a topic in `ros2 topic list` on your laptop, it almost certainly exists on the robot domain; see below.

### 2. Commanding the Drone / Waypoints & Tasks

High-level commands are issued as **ROS 2 actions** under `/{robot}/tasks/{task}`. Because `foxglove_bridge` drops nested fields when calling action services, the GCS does **not** call the actions directly: the Foxglove panels publish a JSON `std_msgs/String` on `…/goal` (and `…/cancel`), and the `action_relay` node parses the JSON into the typed Goal and forwards it to the on-robot action server, streaming `…/relay_feedback` and `…/relay_result` back as JSON Strings. The relay also transforms global-ENU coordinates from the editors into the robot's local `map` frame (subtracting the GPS boot offset) and rejects every non-takeoff task unless the drone is ≥ 5 m AGL.

Tasks: `takeoff`, `land`, `navigate`, `exploration`, `semantic_search`, `fixed_trajectory`.

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/{robot}/tasks/{task}/goal` | `std_msgs/String` (JSON) | Send a task goal from the GCS | Parsed by `action_relay` into the typed action goal. |
| `/{robot}/tasks/{task}/cancel` | `std_msgs/String` | Cancel the active task | Relay forwards a cancel to the robot's action server. |
| `/{robot}/tasks/{task}/relay_feedback` | `std_msgs/String` (JSON) | Live task feedback re-published by the relay | Mirrors the action feedback fields. |
| `/{robot}/tasks/{task}/relay_result` | `std_msgs/String` (JSON) | Final result (`{success, message}`) | Also used to surface rejections (e.g. "takeoff first"). |
| `/{robot}/tasks/{task}` (action) | `task_msgs/action/{Task}Task` | Underlying ROS 2 action server on the robot | e.g. `TakeoffTask`, `LandTask`, `NavigateTask`, `ExplorationTask`, `SemanticSearchTask`, `FixedTrajectoryTask`. |
| `/{robot}/global_plan` | `nav_msgs/Path` | Global waypoint path the local planner follows | Consumed by the navigate task / local planner (remapped to `global_plan`). |

**Goal fields** (from the `action_relay` builders and the `task_msgs` `.action` files):

| Task | Key goal fields |
|---|---|
| `takeoff` | `target_altitude_m` (float, absolute target altitude; must be > 0), `velocity_m_s` (float; 0 = use config default) |
| `land` | `velocity_m_s` (float; 0 = use config default) |
| `navigate` | `global_plan` (`nav_msgs/Path`), `goal_tolerance_m` (float, default 1.0) |
| `exploration` | `search_bounds` (`geometry_msgs/Polygon`, empty = unbounded), `min/max_altitude_agl`, `min/max_flight_speed`, `time_limit_sec` (0 = no limit) |
| `semantic_search` | `query` (string), `background_queries` (string), `search_area` (`geometry_msgs/Polygon`), `min/max_altitude_agl`, `min/max_flight_speed`, `confidence_threshold` (default 0.95) |
| `fixed_trajectory` | `trajectory_spec` (`airstack_msgs/FixedTrajectory` — type + key/value attributes), `loop` (bool) |

**Foxglove waypoint & polygon editors** — the GCS click-to-place panels (publishing on `/clicked_point`, `geometry_msgs/PointStamped`) feed two collector nodes that maintain editable lists, named saves, and rendered markers. The waypoint editor's `…/path` output is the `nav_msgs/Path` you wire into a `navigate` goal; the polygon editor's vertices feed `exploration`/`semantic_search` areas.

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/clicked_point` | `geometry_msgs/PointStamped` | Click in the Foxglove 3D panel to place a waypoint/vertex | Shared by both editors; each gated by an Enable toggle. |
| `/gcs/waypoints/command` | `std_msgs/String` (JSON) | Edit commands: add/delete/move/reorder/clear/set_altitude/saves | |
| `/gcs/waypoints/list` | `std_msgs/String` (JSON) | Active waypoint list for the panel | Latched. |
| `/gcs/waypoints/path` | `nav_msgs/Path` | Active waypoints as a Path, in global `map` frame | Latched. Use as the `navigate` task `global_plan`. |
| `/gcs/waypoints/markers` | `visualization_msgs/MarkerArray` | Active-editor waypoint markers | Latched. |
| `/gcs/waypoints/save_markers` | `visualization_msgs/MarkerArray` | All named saves rendered, each in its own color | Latched. |
| `/gcs/waypoints/saves` | `std_msgs/String` (JSON) | Saved-route metadata (name, color, count, vertices) | Latched. |
| `/gcs/polygon/command` | `std_msgs/String` (JSON) | Polygon edit commands (same verbs as waypoints) | |
| `/gcs/polygon/list` | `std_msgs/String` (JSON) | Active polygon vertices | Latched. |
| `/gcs/polygon/markers` | `visualization_msgs/MarkerArray` | Active polygon outline (closed loop, red) | Latched. |
| `/gcs/polygon/save_markers` | `visualization_msgs/MarkerArray` | All saved polygons, each in its own color | Latched. |
| `/gcs/polygon/saves` | `std_msgs/String` (JSON) | Saved-polygon metadata | Latched. |

### 3. GCS Visualization Topics

The single `foxglove_visualizer_node` auto-discovers each robot's GPS/odometry/trajectory/plan/VDB topics, translates them from each robot's local `map` frame into the shared global ENU `map` frame using the GPS boot offset, and merges them into one MarkerArray for Foxglove.

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/gcs/robot_markers` | `visualization_msgs/MarkerArray` | Combined per-robot markers (body mesh, name label, axes, local trajectory, global plan, VDB map) in global ENU | One merged array for all discovered robots. |
| `/gcs/{robot}/location` | `sensor_msgs/NavSatFix` | Per-robot GPS rewritten to `frame_id='map'` | Foxglove's Map panel only accepts a fix in the `map` frame; this is the per-robot pin (e.g. `/gcs/robot_1/location`). |
| `/gcs/map_origin/location` | `sensor_msgs/NavSatFix` | Stationary fix at the configured `ORIGIN_LAT/LON` | Published at 1 Hz so the Map panel has a fixed reference point. |
| `/gcs/map_origin/ground_msl` | `std_msgs/Float64` | MSL altitude of map `z = 0` (ground datum) | Latched; set once GPS + odom are both available. |
| `/gcs/sim_ground` | `visualization_msgs/Marker` | Sim overhead-camera image rendered as a textured ground plane (TRIANGLE_LIST) | Sim only; latched, built once from `/sim/overhead/image` + `/sim/overhead/spec`. |

### 4. Sensors & Perception

Raw sensor streams live under `/{robot}/sensors/…`; processed perception products under `/{robot}/perception/…`. Image and point-cloud topics use SENSOR_QOS (BEST_EFFORT).

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/{robot}/sensors/front_stereo/left/image_rect` | `sensor_msgs/Image` | Rectified left stereo image | Input to stereo disparity. |
| `/{robot}/sensors/front_stereo/left/camera_info` | `sensor_msgs/CameraInfo` | Left camera intrinsics/calibration | |
| `/{robot}/sensors/front_stereo/right/image_rect` | `sensor_msgs/Image` | Rectified right stereo image | |
| `/{robot}/sensors/front_stereo/right/camera_info` | `sensor_msgs/CameraInfo` | Right camera intrinsics/calibration | |
| `/{robot}/sensors/front_stereo/right/depth_ground_truth` | `sensor_msgs/Image` | Ground-truth depth (sim only) | Evaluation / optional depth-based world model. |
| `/{robot}/sensors/ouster/point_cloud` | `sensor_msgs/PointCloud2` | Ouster 3D LiDAR point cloud | Feeds mapping (e.g. VDB). |
| `/{robot}/perception/stereo_image_proc/point_cloud` | `sensor_msgs/PointCloud2` | Point cloud computed from stereo disparity | Produced by `stereo_image_proc`. |

### 5. Mapping & Plans

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/{robot}/vdb_mapping/vdb_map_visualization` | `visualization_msgs/Marker` | VDB occupancy map mesh for visualization | Per-robot; georeferenced into `/gcs/robot_markers` by the GCS. |
| `/{robot}/trajectory_controller/trajectory_vis` | `visualization_msgs/MarkerArray` | The trajectory currently being executed by the controller | Rendered as the live trajectory on the GCS. |
| `/{robot}/global_plan` | `nav_msgs/Path` | Global waypoint path the robot is following | Output of global planning / input to the navigate task and local planner. |

### 6. Common / Infrastructure Topics

| Topic | Type | Purpose | Notes |
|---|---|---|---|
| `/clock` | `rosgraph_msgs/Clock` | Simulation time | Present when `use_sim_time` is active. |
| `/tf_static` | `tf2_msgs/TFMessage` | Static transforms (e.g. `world`→`map`, sensor mounts) | Dynamic TF (`/tf`) is broadcast by `odometry_conversion`. |
| `/rosout` | `rcl_interfaces/Log` | Aggregated node logging | |
| `/parameter_events` | `rcl_interfaces/ParameterEvent` | Parameter change notifications | |
| `/gossip/peers` | custom AirStack peer-profile message | Multi-robot peer discovery / gossip payload exchange | Used by the coordination/gossip layer. |

> **Quick reference:**
> - **Drone pose / onboard EKF:** `/{robot}/interface/mavros/local_position/odom` (raw MAVROS) and the canonical `/{robot}/odometry_conversion/odometry` (used by the stack).
> - **EKF health:** `/{robot}/interface/mavros/estimator_status`.
> - **Send waypoints:** build a path with the Foxglove waypoint editor (`/gcs/waypoints/path`), then send a `navigate` goal on `/{robot}/tasks/navigate/goal`.
