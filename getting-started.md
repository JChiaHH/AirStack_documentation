# Getting Started

> File paths in this guide are relative to the **AirStack repo root** (e.g. `~/AirStack`), not this documentation repo.

By the end of this guide you'll have the AirStack autonomy stack running — a robot, a Ground Control Station (GCS), and Isaac Sim — on your machine.

> **Alpha / internal usage.** AirStack is currently in alpha and intended for internal usage. Pulling the prebuilt Docker images requires an **AirLab account** (`andrew.cmu.edu`) for access to the `airlab-docker.andrew.cmu.edu` registry and the Nucleus server. If you don't have one, you can build the images from scratch (see [Step 3](#3-get-the-docker-images)), but some assets may still be gated.

## Requirements

| Component | Requirement |
| --- | --- |
| **OS** | Linux. Primarily tested on **Ubuntu 22.04**. |
| **Disk** | At least **25 GB** free for the Docker images. |
| **GPU** | An **NVIDIA GPU** is required for Isaac Sim. **GeForce RTX 4080 or higher** recommended. |
| **NVIDIA Container Toolkit** | Required — installed for you by the installer in [Step 2](#2-install--setup). |

No high-end GPU? The Microsoft AirSim (legacy) and `simple-sim` profiles are lighter-weight alternatives to Isaac Sim.

## 1. Clone

Clone the repository recursively so its submodules are pulled in as well:

```bash
git clone --recursive -j8 https://github.com/castacks/AirStack.git
cd AirStack
```

## 2. Install & Setup

Install the host dependencies (Docker, docker-compose, and the NVIDIA Container Toolkit), then configure the `airstack` CLI:

```bash
./airstack.sh install
./airstack.sh setup
source ~/.bashrc   # or ~/.zshrc
```

- `./airstack.sh install` installs Docker, docker-compose, and the NVIDIA Container Toolkit.
  Useful flags: `--no-docker`, `--with-wintak`, `--force`.
- `./airstack.sh setup` enables the `airstack` command and sets up keys.
- `source ~/.bashrc` (or `~/.zshrc`) applies the updated `PATH` so the `airstack` command works in your shell.

## 3. Get the Docker Images

You have two options. **Option A is preferred** if you have an AirLab account.

### Option A — Pull prebuilt images (preferred)

Log in to the AirLab registry (enter your Andrew ID and password), then pull:

```bash
docker login airlab-docker.andrew.cmu.edu
airstack image-pull
```

The images are large, so this can take a while.

### Option B — Build from scratch

Building locally requires NVIDIA NGC container access.

First, activate your `docker` group membership and confirm the Docker daemon is reachable. The installer in [Step 2](#2-install--setup) adds you to the `docker` group, but that change doesn't take effect in your current shell until you re-login or run `newgrp docker`. `docker info` should then succeed without `sudo` and without permission errors:

```bash
newgrp docker
docker info
```

Then build the images:

```bash
airstack image-build
```

If you have permission, you can optionally push your built images:

```bash
airstack image-push
```

## 4. Launch

Bring up the full stack — robot, GCS, and Isaac Sim. The Isaac scene configured in your `.env` file plays automatically:

```bash
airstack up
```

## 5. Shut Down

Stop and remove the running containers:

```bash
airstack down
```

## Daily Quick Start

Once installed, this is the everyday loop.

**1. Start the Docker daemon**

```bash
sudo systemctl start docker
```

**2. Check Docker status (optional)**

```bash
docker info
```

**3. Bring up AirStack**

```bash
cd AirStack
airstack up
```

**4. Shut AirStack down**

```bash
airstack down
```

## First-Run Validation

After `airstack up`, open the **Foxglove** visualizer and command the robot through a quick sequence: **Takeoff → Navigate → Explore**. If the drone takes off and responds, your stack is working.

> The official tutorial historically mentions an **RViz** window, but this version of AirStack uses **Foxglove Studio** as the default visualizer. See [Operation](operation.md) for the detailed Foxglove walkthrough.

## Next Steps

- [Architecture](architecture.md) — how the stack is organized (containers, layers, DDS).
- [Operation](operation.md) — Foxglove visualizer, resetting Isaac Sim, inspecting ROS 2 topics.
- [Configuration](configuration.md) — `.env`, multi-robot, swapping drones/sensors, troubleshooting.
