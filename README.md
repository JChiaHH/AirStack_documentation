# AirStack Documentation

Practical operator + developer notes for running and understanding [AirStack](https://github.com/castacks/AirStack) (AirLab, CMU) — a layered autonomous aerial robotics stack on ROS 2 (Jazzy), Docker, and Isaac Sim (Pegasus + PX4 SITL).

> File paths throughout these docs are relative to the **AirStack repo root** (e.g. `~/AirStack`), not this documentation repo.

These notes complement the official docs at <https://docs.theairlab.org/main/docs/getting_started/>. They were originally one long page and have been split into focused guides:

## Documentation Map

| Guide | Read it when you want to… |
|---|---|
| **[Getting Started](getting-started.md)** | Install AirStack from scratch — requirements, clone, `airstack install` / `setup`, pull or build images, first launch and validation. |
| **[Architecture](architecture.md)** | Understand how the stack fits together — the container layout, the layered autonomy pipeline, the multi-domain DDS bridge, and the GCS / drone / sensor models. |
| **[Operation](operation.md)** | Run and inspect a live stack — open the Foxglove visualizer, reset Isaac Sim without a full restart, and find/echo ROS 2 topics across DDS domains (full topic reference). |
| **[Configuration & Troubleshooting](configuration.md)** | Change what runs — `.env` knobs, multi-robot setup, swapping drones/sensors and exposing new topics, plus fixes for the most common failures. |

## What is AirStack?

AirStack is organized as a set of Docker containers on one bridge network, running a layered ROS 2 autonomy stack:

```
Sensors → Perception → World Models → Planners → Controllers → Interface → Hardware
```

Each robot runs on its own DDS domain (`ROS_DOMAIN_ID`), and a cross-domain bridge surfaces fleet state to a **Foxglove-based Ground Control Station** (the default visualizer in this version — not RViz). Multi-robot is achieved with Docker Compose replicas, not in-container namespacing. See [Architecture](architecture.md) for the full picture.

## TL;DR — Daily Use

Once installed (see [Getting Started](getting-started.md)):

```bash
sudo systemctl start docker   # ensure the Docker daemon is running
cd AirStack
airstack up                   # launch robot + GCS + Isaac Sim
# ... open Foxglove and command Takeoff → Navigate → Explore ...
airstack down                 # stop and remove containers
```

For the Foxglove walkthrough and topic inspection, see [Operation](operation.md). If takeoff is rejected or nothing shows in Foxglove, see [Troubleshooting](configuration.md#troubleshooting).
