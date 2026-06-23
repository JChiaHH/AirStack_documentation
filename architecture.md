# AirStack Architecture

> File paths in this document are relative to the **AirStack repo root** (e.g. `~/AirStack`), not this documentation repo.

[← Docs index](README.md) · [Getting Started](getting-started.md) · [Operation](operation.md) · [Configuration](configuration.md)

AirStack is organized as a set of Docker containers wired onto one custom bridge network, running a layered ROS 2 (Jazzy) autonomy stack. Each robot lives on its own DDS domain, and a cross-domain bridge surfaces fleet state to a Foxglove-based Ground Control Station.

## Component Containers

The top-level `docker-compose.yaml` includes one compose file per component and defines the shared bridge network:

| Service | Compose file | Role |
|---|---|---|
| **robot** (`robot-desktop`, plus `-onboard`/`-offboard`, `robot-voxl`, `robot-l4t`) | `robot/docker/docker-compose.yaml` | The onboard ROS 2 Jazzy autonomy stack. `robot-desktop` is the dev/sim profile (x86 + GPU, launches `desktop_bringup` with `AUTONOMY_ROLE=full`). Scaled to multiple robots via `deploy.replicas: ${NUM_ROBOTS}`. Variants cover Jetson (`robot-l4t`), VOXL (`robot-voxl`), and split onboard/offboard deployments. |
| **isaac-sim** | `simulation/isaac-sim/docker/docker-compose.yaml` | NVIDIA Isaac Sim with the Pegasus extension and PX4 SITL. Pinned to the static address `172.31.0.200` on the bridge network so robots can reach it deterministically. Runs a launch script (e.g. `example_one_px4_pegasus_launch_script.py`). |
| **gcs** (`gcs`, `gcs-real`) | `gcs/docker/docker-compose.yaml` | Ground Control Station. Runs the GCS bringup (`gcs.launch.xml`): `foxglove_bridge`, the GCS visualizer, and the action relay. Operates on `ROS_DOMAIN_ID=0` and bridges to each robot's domain. |
| **docs** | `docs/docker/docker-compose.yaml` | Live MkDocs documentation server on port `8000`. |

All services attach to the **custom bridge network `airstack_network` (subnet `172.31.0.0/24`)** defined in the top-level compose file, isolating inter-container traffic from other Docker networks on the host. (Field-deployment profiles such as `gcs-real`, `robot-l4t`, and `robot-voxl` instead use `network_mode: host`.)

## Layered Autonomy Stack

The onboard stack under `robot/ros_ws/src/` follows a strict layered data flow. Each layer is a directory containing individual algorithm modules plus a `*_bringup` package that wires them together with topic remapping.

The data path, in one line:

```
Sensors → Perception → World Models → Planners → Controllers → Interface → Hardware
```

Read top-to-bottom, the loop looks like this — **Behavior** sits above and drives the chain, while data flows down from the flight controller and commands flow back to it:

```
   ┌────────────────────────────────────────────────────────────────┐
   │  BEHAVIOR  (orchestration)                                      │
   │  behavior_tree · task executors  — drives the loop below        │
   └───────────────────────────────┬────────────────────────────────┘
                                    │ goals / tasks
                                    ▼
   ┌─────────────┐ ◀───────────────────────────────────────────┐
   │  HARDWARE   │   PX4 / flight controller (or PX4 SITL)      │
   └──────┬──────┘                                              │
          │ raw measurements                                    │
          ▼                                                     │
   ┌─────────────┐                                              │
   │  SENSORS    │   lidar_point_cloud_filter, stereo           │
   └──────┬──────┘                                              │
          ▼                                                     │
   ┌─────────────┐                                              │
   │ PERCEPTION  │   macvo_ros2  (state estimation / VIO)       │
   └──────┬──────┘                                              │
          ▼                                                     │
   ┌─────────────┐   local:  disparity_expansion                │
   │ WORLD MODELS│   global: vdb_mapping_ros2                    │
   └──────┬──────┘                                              │
          ▼                                                     │
   ┌─────────────┐   local:  droan_local_planner                │
   │  PLANNERS   │   global: random_walk                         │
   └──────┬──────┘                                              │
          ▼                                                     │
   ┌─────────────┐                                              │
   │ CONTROLLERS │   trajectory_controller                      │
   └──────┬──────┘                                              │
          ▼                                                     │
   ┌─────────────┐   mavros_interface / px4_interface (+ safety)│
   │  INTERFACE  │ ─────────────────────────────────────────────┘
   └─────────────┘   sends control commands back to Hardware
```

Each arrow is one or more ROS 2 topics, remapped inside the `*_bringup` packages so algorithm implementations can be swapped without touching downstream layers.

| Layer | What it does | Source dir / reference module |
|---|---|---|
| **Sensors** | Acquires raw sensor data (LiDAR, stereo, gimbal) and publishes it on standard topics | `sensors/` — e.g. `lidar_point_cloud_filter` |
| **Perception** | State estimation and visual processing (e.g. visual odometry) | `perception/` — `macvo_ros2` |
| **World Models** | Builds local and global obstacle/occupancy representations | local: `local/world_models/disparity_expansion`, global: `global/world_models/vdb_mapping_ros2` |
| **Planners** | Local (reactive) and global (mission-scale) path planning | local: `local/planners/droan_local_planner`, global: `global/planners/random_walk` |
| **Controllers** | Converts planned trajectories into control commands | `local/controls/trajectory_controller` (also `attitude_controller`, `pid_controller`) |
| **Interface** | Hardware/flight-controller bridge (MAVROS/PX4) and safety | `interface/` — `mavros_interface`, `px4_interface`, `robot_interface` |
| **Hardware** | Flight controller / vehicle (or PX4 SITL in Isaac Sim) | PX4 via the interface layer |
| **Behavior** (orchestration) | High-level mission execution driving the layers above via behavior trees and task executors | `behavior/` — `behavior_tree` |

The bringup variant is selected by `AUTONOMY_ROLE` (`full` | `onboard` | `offboard`) in `autonomy_bringup/launch/robot.launch.xml`.

## Multi-Domain / DDS Architecture

Each robot runs on its **own `ROS_DOMAIN_ID`** so robots are isolated DDS partitions by default — **robot N → domain N**. Multi-robot is achieved with Docker Compose replicas (`NUM_ROBOTS=3` → `airstack-robot-desktop-1/-2/-3`), not in-container namespacing.

**Name + domain resolution at startup.** Each container computes its identity in `robot/docker/.bashrc`: it reads `ROBOT_NAME_SOURCE` (`container_name` or `hostname`) to obtain a name, then runs `robot/docker/robot_name_map/resolve_robot_name.py` against the YAML mapping (e.g. `default_robot_name_map.yaml`). The resolver regex-matches the container/host name and exports both `ROBOT_NAME` (e.g. `robot_1`) and `ROS_DOMAIN_ID` (e.g. `1`). An explicit `ROS_DOMAIN_ID` from the environment takes precedence — this is how offboard GCS containers are forced onto domain 0.

**Crossing domains to the GCS (domain 0).** Robot topics are pinned to the robot's domain; the GCS lives on domain 0. Three mechanisms bridge them:

1. **DDS router allowlist** — `robot/ros_ws/src/autonomy_bringup/onboard_all/config/dds_router.yaml` defines two participants (`robot` on `$(env ROS_DOMAIN_ID)` and `gcs` on `$(var gcs_domain)`) and an explicit **allowlist** of topics/services that may cross (odometry, GPS, trajectory vis, VDB map, camera/LiDAR streams, behavior-tree services). Only allowlisted topics traverse domains; the router re-reads its allowlist only at startup. (Gossip peer profiles are bridged separately on domain 99.)
2. **action_relay** — `gcs/ros_ws/src/action_relay` bridges *actions* (not just topics). Because `foxglove_bridge` drops nested fields when calling action services, the relay subscribes to plain `String`/JSON command topics on domain 0, rebuilds typed `Goal` messages, and forwards them via an `ActionClient` on the robot's domain N — republishing feedback/result back on domain 0. One relay node is spawned per robot. It also gates non-takeoff tasks on an "airborne" precondition and re-references global-ENU waypoints to each robot's GPS boot offset.
3. **GCS Foxglove visualizer** — `gcs/ros_ws/src/gcs_visualizer` (`foxglove_visualizer_node`) runs on domain 0 and subscribes to the bridged per-robot topics. Every 5 s it auto-discovers robots by regex-matching the topic prefix, then merges each robot's markers (mesh, trajectory, global plan, VDB map) into a single `/gcs/robot_markers` `MarkerArray`, translating each robot's local `map` frame into one global ENU frame using its GPS boot offset.

The bridge, drawn top-to-bottom (robot domain N at the top, operator at the bottom):

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  ROBOT N                                   ROS_DOMAIN_ID = N      │
  │                                                                   │
  │   Autonomy topics                     Task executors              │
  │   odometry · GPS · sensors            (ROS 2 ActionServers)       │
  │   traj_vis · VDB map                            ▲                 │
  └───────────┬─────────────────────────────────────┼────────────────┘
              │ allowlisted topics (N → 0)           │ typed Goal
              ▼                                       │ (ActionClient, 0 → N)
  ┌───────────┴───────────────────────────────────────┴──────────────┐
  │  CROSS-DOMAIN BRIDGE                                              │
  │   • DDS router    — dds_router.yaml allowlist  (copies N → 0)     │
  │   • action_relay  — rebuilds JSON commands into typed Goals       │
  └───────────┬─────────────────────────────────────▲────────────────┘
              │ bridged topics                       │ task goal
              ▼                                       │ (JSON String)
  ┌───────────┴───────────────────────────────────────┴──────────────┐
  │  GCS                                       ROS_DOMAIN_ID = 0       │
  │                                                                   │
  │   foxglove_visualizer_node ──▶ /gcs/robot_markers                 │
  │   foxglove_bridge          ──▶ ws://localhost:8765                │
  │                         ▲   │                                     │
  │                         │   ▼                                     │
  │                    Foxglove Studio   (operator view + commands)   │
  └───────────────────────────────────────────────────────────────────┘

  Telemetry  :  robot topics ──▶ DDS router ──▶ visualizer ──▶ Foxglove
  Commanding :  Foxglove goal ──▶ action_relay ──▶ robot ActionServer
  Feedback   :  ActionServer ──▶ action_relay ──▶ Foxglove
```

## GCS / Visualization

**Foxglove Studio is the default visualizer in this version (not RViz** — the RViz/rqt nodes in `gcs.launch.xml` are wrapped in `<?ignore ?>` blocks). The GCS bringup launches:

- **`foxglove_bridge`** on port **8765** (`address 0.0.0.0`), exposing domain-0 topics over WebSocket. Connect at `ws://localhost:8765` inside the container, or `ws://localhost:8766` from the host (port-mapped in the gcs compose service).
- **`foxglove_visualizer_node`** plus the waypoint/polygon collector and gossip-payload nodes, which produce the georeferenced fleet view and interactive editors.

A **`NUM_ROBOTS`-matched Foxglove layout is auto-generated on every container startup** by `gcs/foxglove_extensions/render_layout.py`, which expands the single-robot template `airstack_default.json` into `/root/airstack_layout_num_robots_<N>.json` (regenerated each boot, removed with the container). Import that file once per `NUM_ROBOTS` change. See [Open the Visualizer in Foxglove](operation.md#open-the-visualizer-in-foxglove).

---

## Ground Control Station, Drone & Sensors

### Ground Control Station (GCS)

The **Ground Control Station** is the operator-side half of AirStack — the software you use to *watch* and *command* the fleet, as opposed to the autonomy that runs *on* each drone. It runs in its own `gcs` container on `ROS_DOMAIN_ID=0` and is the only place a human (or an external client) interacts with the robots.

The GCS does **not** fly the drone itself; the onboard autonomy stack does that. The GCS's job is to bridge the operator and the per-robot DDS domains, and to provide visualization and high-level command tools. It runs:

- **Foxglove Studio + `foxglove_bridge`** (port 8765) — the visualization front-end. This is the default visualizer in this version of AirStack (RViz is disabled). See [Open the Visualizer in Foxglove](operation.md#open-the-visualizer-in-foxglove).
- **`foxglove_visualizer_node`** — discovers each robot's bridged topics and merges them into one georeferenced fleet view (`/gcs/robot_markers`) in a shared global ENU frame.
- **`action_relay`** — translates the JSON commands sent from Foxglove panels into typed ROS 2 action goals on each robot's domain (takeoff, navigate, etc.), and relays feedback/results back.
- **Waypoint & polygon editors** — interactive click-to-place tools for building routes and geofence/search areas.

Because each robot lives on its own DDS domain, the GCS reaches them through the cross-domain bridge (DDS router + action relay) described in [Multi-Domain / DDS Architecture](#multi-domain--dds-architecture). A field variant, `gcs-real`, runs the same software in host-network mode for deployment on a real ground laptop.

### Drone Model

The default simulated vehicle is the **Pegasus Simulator Iris quadrotor** running **PX4 SITL**.

| Property | Value |
|---|---|
| Airframe | **Iris quadrotor** (4 rotors), Pegasus Simulator asset |
| Autopilot / firmware | **PX4 SITL** (auto-launched by Pegasus, MAVLink lockstep) |
| Simulator | NVIDIA Isaac Sim + **Pegasus Simulator** extension |
| Drone USD (sim model) | Pegasus `assets/Robots/Iris/iris.usd` |
| ROS URDF (frames/TF) | `common/ros_packages/robot_descriptions/iris/urdf/iris_with_sensors.pegasus.robot.urdf` (set via `URDF_FILE` in `.env`) |
| MAVLink port | `14540 + vehicle_id` (e.g. `14541` for robot_1) |
| Default launch script | `example_one_px4_pegasus_launch_script.py` (single drone) |

The drone's kinematic tree (from the URDF) is: `base_link` → body → 4 rotors, plus a LiDAR mount (`ouster`) and a ZED camera body (`camera_left` / `camera_right` / `imu`). To fly a different airframe you change the drone USD in the launch script **and** the `URDF_FILE` — see [Adding / Changing a Drone or Sensors](configuration.md#adding--changing-a-drone-or-sensors).

### Onboard Sensors

Two of the drone's sensors are **simulated by Isaac Sim** (the ZED stereo camera and the Ouster LiDAR); the rest (IMU, GPS, barometer/altitude) come from **PX4's flight controller via MAVROS**, not from a separate ROS driver. Sensor mounting offsets and configs are set in the launch script.

| Sensor | Model / Config | Measures | ROS 2 topic(s) | Type | Notes |
|---|---|---|---|---|---|
| **Stereo camera** | Stereolabs **ZED X** (Isaac asset) | Rectified RGB (left & right) + ground-truth depth | `…/sensors/front_stereo/{left,right}/image_rect`, `…/{left,right}/camera_info`, `…/right/depth_ground_truth` | `sensor_msgs/Image`, `CameraInfo` | Mounted forward, offset `[0.2, 0.0, −0.05]` m. Default resolution **480×300** (extension default). `depth_ground_truth` is **sim-only** ground truth, not a real stereo depth estimate. |
| **LiDAR** | **Ouster OS1** (`ouster_os1`; 128-ch, 10 Hz, 512 res) | 3D point cloud | `…/sensors/ouster/point_cloud` (filtered; raw is `…/point_cloud_raw`) | `sensor_msgs/PointCloud2` | Offset `[0.0, 0.0, 0.025]` m, `min_range = 0.75` m. **Always enabled** in the single-drone script; in the *multi*-drone script it is gated by `ENABLE_LIDAR`. |
| **IMU (flight controller)** | PX4 SITL EKF | Body acceleration, angular rate, orientation | `…/interface/mavros/imu/data` | `sensor_msgs/Imu` | From **PX4 via MAVROS** (robot domain only). This is the IMU the EKF fuses. |
| **GPS** | PX4 SITL (simulated GPS → EKF) | Global position (lat/lon/alt) | `…/interface/mavros/global_position/global` | `sensor_msgs/NavSatFix` | From **PX4 EKF via MAVROS**. Bridged to the host/GCS domain. The GCS uses the first fix as the robot's ENU boot origin. |
| **Barometer / altitude** | PX4 SITL | Altitude (AMSL / relative / terrain) | `…/interface/mavros/altitude` | `mavros_msgs/Altitude` | From **PX4 EKF via MAVROS** (robot domain only). |

> **Note — there is also a ZED *camera* IMU frame** (`imu` link under the ZED body) in the URDF, but the default launch graph does not publish it as a ROS topic — only the flight-controller IMU above is live. The `/sim/overhead/image` topic you may see is a **world/scene overhead camera for visualization, not a sensor on the drone.**

> Most onboard-sensor topics are **robot-domain-only** (not visible from your host terminal by default) — see [Viewing topics across DDS domains](operation.md#viewing-topics-across-dds-domains).
