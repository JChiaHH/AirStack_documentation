# Configuration & Troubleshooting

> File paths in this document are relative to the **AirStack repo root** (e.g. `~/AirStack`), not this documentation repo.

[← Docs index](README.md) · [Getting Started](getting-started.md) · [Architecture](architecture.md) · [Operation](operation.md)

## Configuration Files

AirStack is configured almost entirely through a single top-level `.env` file (consumed by Docker Compose for variable interpolation) plus a small set of per-component config files.

| Config File (path) | Purpose | Multi-robot? | Different drone/sensors? |
|---|---|---|---|
| `.env` | Master env file. Sets Docker Compose interpolation vars (image tags, profiles, robot count, Isaac Sim launch mode/script, URDF, ports). | **Yes** — set `NUM_ROBOTS`, switch `ISAAC_SIM_SCRIPT_NAME` to the multi script. | **Yes** — point `URDF_FILE` and `ISAAC_SIM_GUI` at the right model/scene. |
| `docker-compose.yaml` | Top-level compose; `include:`s all component compose files and defines the `airstack_network` bridge (`172.31.0.0/24`). | Rarely | No |
| `robot/docker/docker-compose.yaml` | Defines all robot services. Implements `deploy.replicas: ${NUM_ROBOTS:-1}` and per-service `AUTONOMY_ROLE` + `LAUNCH_PACKAGE`. | **Yes (indirectly)** — `replicas` driven by `NUM_ROBOTS`; ports already support up to 21 robots. | Only when adding a new hardware/sensor-driver service. |
| `robot/docker/robot_name_map/default_robot_name_map.yaml` | Regex rules mapping container/host name → `ROBOT_NAME` and `ROS_DOMAIN_ID` (first match wins). | **Usually no** — default rule auto-assigns `robot_<N>`/domain `<N>`. | No |
| `robot/docker/robot_name_map/resolve_robot_name.py` | Applies the YAML rules at startup; selects input via `ROBOT_NAME_SOURCE` (`container_name`/`hostname`). | No (mechanism) | No |
| `robot/ros_ws/src/autonomy_bringup/onboard_all/config/dds_router.yaml` | Cross-domain allowlist (eProsima DDS Router) bridging robot domain ↔ GCS domain. Topics namespaced `rt/$(env ROBOT_NAME)/...`. | No — templated per robot. | **Yes** — add a line per new topic to expose to the GCS. |
| `robot/ros_ws/src/autonomy_bringup/onboard_local_offboard_global/config/dds_router.yaml` | Split-config variant; `extends` `onboard_all` and adds onboard/offboard bridge topics. | No | **Yes** (split onboard/offboard role) |
| `robot/ros_ws/src/autonomy_bringup/onboard_local_offboard_global/config/domain_bridge.yaml` | `domain_bridge` config for the split; relays `global_plan` from `gcs_domain` to the robot's domain. | No | Only if new topics must cross the onboard/offboard boundary. |
| `simulation/isaac-sim/launch_scripts/example_one_px4_pegasus_launch_script.py` | **Single-drone** Isaac Sim/Pegasus launcher. Hard-codes `robot_1`, `vehicle_id=1`, `domain_id=1`, ZED camera + RTX Ouster lidar. | **No — use the multi script.** | **Yes** — edit `DRONE_USD`, camera/lidar offsets, `lidar_config`. |
| `simulation/isaac-sim/launch_scripts/example_multi_px4_pegasus_launch_script.py` | **Multi-drone** launcher parametrized by `NUM_ROBOTS`. Loops `spawn_drone(i)` → `robot_<i>`, `vehicle_id=i`, `domain_id=i`. Honors `ENABLE_LIDAR`, `ISAAC_SIM_HEADLESS`, `PLAY_SIM_ON_START`. | **Yes — set `ISAAC_SIM_SCRIPT_NAME` to this.** | **Yes** — same drone/sensor knobs as single script. |
| `gcs/foxglove_extensions/airstack_default.json` | Foxglove layout **template** (canonical `robot_1` tab). | Edit only to change the per-robot layout. | Add panels/topics here for new sensors. |
| `gcs/foxglove_extensions/render_layout.py` | Regenerates the layout with one tab per robot from the template, driven by `NUM_ROBOTS`. Runs at GCS startup → `/root/airstack_layout_num_robots_<N>.json`. | **Yes (automatic)** — re-import the generated file. | No |
| `gcs/docker/gcs-base-docker-compose.yaml` | GCS container; runs `install.py` then `render_layout.py` at startup; sets `NUM_ROBOTS`. | Inherits `NUM_ROBOTS` automatically. | No |
| `robot/ros_ws/src/autonomy_bringup/launch/robot.launch.xml` | Single entry point for all robot deployments. Dispatches on `role` (`AUTONOMY_ROLE`); pushes `$(env ROBOT_NAME)` namespace; loads URDF via `URDF_FILE`; wires DDS router + gossip. | No (templated) | **Yes** — `urdf_file` defaults to `$(env URDF_FILE)`. |
| `robot/ros_ws/src/behavior/behavior_bringup/launch/behavior.launch.xml` | Behavior layer bringup; runs `drone_safety_monitor` with tunable `state_estimate_timeout` (default `1.0` s) watchdog on odometry. | No | Tune `state_estimate_timeout` if state estimate runs at a different rate. |
| `robot/ros_ws/src/interface/interface_bringup/launch/px4_config.yaml` | MAVROS/PX4 interface config. | No | **Yes** if the flight controller / MAVLink setup differs. |
| `common/ros_packages/robot_descriptions/iris/urdf/iris_with_sensors.pegasus.robot.urdf` | ROS-side URDF (default `URDF_FILE`) describing links + sensor frames for `robot_state_publisher`/TF. | No | **Yes** — edit/replace to change the model or sensor frames. |

### Key `.env` Variables

| Variable | Example value | What it does |
|---|---|---|
| `PROJECT_NAME` | `airstack` | Docker repo name / image tag component. |
| `VERSION` | `0.18.0` | Image tag version. |
| `DOCKER_IMAGE_BUILD_MODE` | `dev` | `dev` = mounted code built live; `prebuilt` = ros_ws baked into image. |
| `PROJECT_DOCKER_REGISTRY` | `airlab-docker.andrew.cmu.edu/airstack` | Where images are pushed/pulled. |
| `COMPOSE_PROFILES` | `desktop,isaac-sim` | Which services start. Others: `desktop_split`, `offboard`, `simple`, `voxl`, `l4t`, `hitl`, `deploy`, `test`. |
| `AUTOLAUNCH` | `true` | If `false`, containers spawn idle with no launch command. |
| `NUM_ROBOTS` | `1` | Number of robot containers (`deploy.replicas`), Foxglove tabs, and sim drones. |
| `RECORD_BAGS` | `false` | Toggle rosbag recording. |
| `ISAAC_SIM_GUI` | `.../scenes/simple_pegasus.scene.usd` | Scene USD loaded when not using a standalone script. |
| `ISAAC_SIM_USE_STANDALONE` | `true` | If `true`, launch Isaac Sim via a standalone Python script instead of opening the USD. |
| `ISAAC_SIM_SCRIPT_NAME` | `example_one_px4_pegasus_launch_script.py` | Which script in `simulation/isaac-sim/launch_scripts/` to run. **Switch to `example_multi_px4_pegasus_launch_script.py` for multi-robot.** |
| `PLAY_SIM_ON_START` | `false` | Auto-play the sim timeline on launch. |
| `ROBOT_NAME_MAP_CONFIG_FILE` | `default_robot_name_map.yaml` | Which YAML resolves `ROBOT_NAME` / `ROS_DOMAIN_ID`. |
| `URDF_FILE` | `robot_descriptions/iris/urdf/iris_with_sensors.pegasus.robot.urdf` | URDF loaded by `robot.launch.xml`. **Change for a different drone.** |
| `DEBUG_RVIZ` | `false` | If `true`, launches RViz alongside the robot. |
| `OFFBOARD_BASE_PORT` / `ONBOARD_BASE_PORT` | `14540` / `14580` | Base MAVLink ports; incremented per agent so multi-agent FCU ports don't collide. |

> **Note:** `ROBOT_NAME` and `ROS_DOMAIN_ID` are **not** set in `.env`; they are resolved per container by `resolve_robot_name.py`. Some knobs (`ENABLE_LIDAR`, `ISAAC_SIM_HEADLESS`, `AUTONOMY_ROLE`, `ROBOT_NAME_SOURCE`) are read as container environment variables / set per service in `robot/docker/docker-compose.yaml` rather than appearing in `.env` by default — add them to `.env` if you want to override the defaults.

### Multi-Robot Configuration

1. **Set the count** in `.env`:
   ```
   NUM_ROBOTS="3"
   ```
   This drives `deploy.replicas: ${NUM_ROBOTS}` (N robot containers) and `NUM_ROBOTS` in the GCS.

2. **Switch the sim launch script** to the multi-drone version:
   ```
   ISAAC_SIM_SCRIPT_NAME="example_multi_px4_pegasus_launch_script.py"
   ```
   It loops `spawn_drone(i)` for `i = 1..NUM_ROBOTS`, assigning `robot_<i>`, `vehicle_id=i`, `domain_id=i`, and spacing them along X. (Optionally set `ENABLE_LIDAR=true` — the multi script defaults lidar off.)

3. **Robot names / domain IDs** are automatic. The default rule in `default_robot_name_map.yaml`:
   ```yaml
   - pattern: '.*robot-.*(\d+)'
     robot: 'robot_{1}'
     domain_id: '{1}'
   ```
   yields `robot_1/2/3...` with matching domain IDs. Edit only for a custom scheme.

4. **DDS router / domain bridge** need no per-robot edits — allowlist entries are templated with `$(env ROBOT_NAME)` / `$(env ROS_DOMAIN_ID)`.

5. **Foxglove layout** regenerates automatically on GCS startup (`render_layout.py`). Re-import `/root/airstack_layout_num_robots_<N>.json` in Foxglove.

6. **Ports** already accommodate up to 21 robots — no change needed for typical fleet sizes.

### Adding / Changing a Drone or Sensors

**Sim drone/model and sensors** are defined in the launch script (`example_one_…` / `example_multi_…`):
- `DRONE_USD` — path to the drone USD asset (default Pegasus Iris).
- `add_zed_stereo_camera_subgraph(...)` — `camera_offset`, `camera_rotation_offset`, `camera_name`.
- `add_rtx_lidar_subgraph(...)` — `lidar_config` (e.g. `ouster_os1`), `lidar_topic_name`, `lidar_offset`, `min_range` (gated on `ENABLE_LIDAR` in the multi script).
- `init_pos` / `init_orient` — spawn pose; `vehicle_id` sets the MAVLink port (`14540 + vehicle_id`).

Deeper spawn behavior lives in the Pegasus OmniGraph APIs under `simulation/isaac-sim/extensions/PegasusSimulator/.../ogn/api/` (`spawn_multirotor.py`, `spawn_zed_camera.py`, `spawn_rtx_lidar.py`).

**ROS-side description:** update `URDF_FILE` in `.env` and the corresponding URDF under `common/ros_packages/robot_descriptions/` so `robot_state_publisher` and TF reflect the new links / sensor frames.

**Hardware sensor drivers (real robot):** add/adjust a driver service in `robot/docker/docker-compose.yaml` (see `zed-l4t` as the pattern); for the flight controller, edit `interface/interface_bringup/launch/px4_config.yaml`.

**Exposing a new topic to the GCS:** add a line to the `dds_router.yaml` allowlist:
- topic: `- name: "rt/$(env ROBOT_NAME)/sensors/<new_sensor>/<topic>"`
- service: both `rq/...Request` and `rr/...Reply` entries.

For the split onboard/offboard config, also add it to `onboard_local_offboard_global/config/dds_router.yaml` and (if it must cross domains) `domain_bridge.yaml`. To show the new sensor in Foxglove, add the panel/topic to `gcs/foxglove_extensions/airstack_default.json` (propagates to all tabs on the next `render_layout.py` run).

### Files You Typically CREATE New

- A new Isaac Sim launch script in `simulation/isaac-sim/launch_scripts/` (clone an existing one), then set `ISAAC_SIM_SCRIPT_NAME`.
- A new URDF under `common/ros_packages/robot_descriptions/<model>/urdf/`, then point `URDF_FILE` at it.
- A new robot-name-map YAML in `robot/docker/robot_name_map/`, then set `ROBOT_NAME_MAP_CONFIG_FILE`.
- A new scene USD under `simulation/isaac-sim/assets/scenes/`, then set `ISAAC_SIM_GUI`.
- A new sensor-driver service block in `robot/docker/docker-compose.yaml` for new real hardware.

### Files You Typically EDIT In Place

- `.env` — the primary knob board (`NUM_ROBOTS`, `ISAAC_SIM_SCRIPT_NAME`, `URDF_FILE`, `COMPOSE_PROFILES`, `ISAAC_SIM_GUI`).
- `simulation/isaac-sim/launch_scripts/example_*_px4_pegasus_launch_script.py` — drone USD, camera/lidar offsets and config.
- `robot/ros_ws/src/autonomy_bringup/onboard_all/config/dds_router.yaml` — add bridged topics/services.
- `gcs/foxglove_extensions/airstack_default.json` — the layout template.
- `robot/ros_ws/src/behavior/behavior_bringup/launch/behavior.launch.xml` — `state_estimate_timeout`.
- `robot/docker/robot_name_map/default_robot_name_map.yaml` — only for custom naming/domain rules.
- `robot/ros_ws/src/interface/interface_bringup/launch/px4_config.yaml` — flight-controller/MAVLink tuning.

---

## Troubleshooting

### Takeoff goal is rejected ("Robot rejected goal" / "state estimate timed out")

The takeoff action server rejects a goal at its precondition check if it sees `state_estimate_timed_out = true`. That flag is published by `drone_safety_monitor`, which sets it when no odometry arrives on `/{robot}/odometry_conversion/odometry` within `state_estimate_timeout` (default 1.0 s).

Causal chain to check, top-down:
1. **No odometry** on `/{robot}/odometry_conversion/odometry` → its input `…/mavros/local_position/odom` is empty →
2. **PX4 not producing telemetry.** In sim this is usually a dead PX4 — check for a `<defunct>` PX4 process:
   ```bash
   docker exec isaac-sim bash -lc 'pgrep -af px4'
   ```
   A pymavlink error in the Isaac Sim window (`AttributeError: 'NoneType' object has no attribute 'recv_match'`) confirms the Pegasus↔PX4 MAVLink bridge lost its connection.

**Fix:** restart the sim (see [Reset Isaac Sim Without Restarting AirStack](operation.md#reset-isaac-sim-without-restarting-airstack)), then confirm odometry is flowing before commanding takeoff. Note that `target_altitude_m` must be `> 0` and the drone must not already have an active task — the other two rejection reasons.

### Nothing appears in Foxglove

- Confirm the data source is connected (top bar shows `ws://localhost:8765`, not "No data source"). Re-open the connection if needed.
- Confirm a layout is loaded (import `airstack_layout_num_robots_<N>.json`).
- Other tasks (navigate/explore/etc.) are rejected unless the drone is **≥ 5 m AGL** — take off first.
