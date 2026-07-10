# Mapless RL Navigation with TurtleBot3

This repository contains the code and simulation environments for a **Mapless Reinforcement Learning navigation system** using a TurtleBot3 in ROS 2 and Gazebo. Training uses a **Soft Actor-Critic (SAC)** algorithm with **LSTM** for continuous control in both static and dynamic scenarios.


---

## Table of Contents

1. [Key Features](#key-features)
2. [Benchmark Results](#benchmark-results)
3. [System Architecture](#system-architecture)
4. [Project Structure](#project-structure)
5. [Getting Started](#getting-started)
6. [Automated Benchmark System](#automated-benchmark-system)
7. [Running Multiple Experiments in Parallel](#running-multiple-experiments-in-parallel)
8. [Automatic Container Isolation](#automatic-container-isolation)
9. [Development Environment Setup](#development-environment-setup)
10. [Prerequisites & Installation](#prerequisites--installation)
11. [Troubleshooting](#troubleshooting)

---

## Key Features

- **SAC + LSTM** continuous control for mapless navigation in static & dynamic environments
- **Fully isolated DevContainers** with automatic Zenoh + Gazebo port assignment (run many experiments in parallel, zero manual config)
- **Automated benchmark framework** — runs multiple RL variants (PER, uniform buffer, MLP baseline) each in its own container
- **Git Worktree-based experiment management** for reproducible parallel training
- **Auto GPU detection** — installs the correct PyTorch CUDA wheel per machine (sm_86 / sm_89 / sm_120)
- **Headless training** with safe live monitoring via `gzclient` and ROS2 topics

---

## Benchmark Results

### Gazebo Navigation Results

SAC + LSTM evaluation across TurtleBot3 stages (progressive difficulty), including the final dynamic-obstacle stage. Each stage was evaluated over 50 test episodes (turtlebot_world over 100).The current codebase and the displayed results may not fully correspond, as the implementation is still being updated. The results shown here represent the best outcome based on the old state of the code, while the logic is actively being refined for better generalization

| Metric | turtlebot_world | Stage 1 | Stage 2 | Stage 3(Fixed-orientation Moving Obs.) | Stage 4  | Stage 4 (Dynamic Moving Obs.) |
|--------|:---------------:|:-------:|:-------:|:-------:|:--------------------------------------:|:-----------------------------:|
| Total Episodes       | 100     | 50      | 50     | 50      | 50     | 50     |
| Successes (Goal)     | 78      | 50      | 50     | 23      | 50     | 40     |
| **Success Rate (%)** | **78%** | **100%**| **100%**| **46%**| **100%**| **80%** |
| Collisions           | 22      | 0       | 0      | 27      | 0      | 10     |
| Truncated            | 0       | 0       | 0      | 0       | 0      | 0      |
| Reward (Min)         | -67.81  | 59.12   | 55.47  | -55.22  | 54.89  | -54.38 |
| Reward (Max)         | 70.52   | 100.56  | 87.58  | 72.18   | 83.73  | 77.84  |
| Reward (Mean All)    | 34.62   | 76.72   | 70.39  | 2.66    | 66.73  | 44.02  |
| Steps (Min)          | 49      | 64      | 32     | 48      | 36     | 37     |
| Steps (Max)          | 364     | 777     | 493    | 314     | 441    | 601    |
| Steps (Mean Succ.)   | 166.4   | 286.4   | 245.1  | 138.17  | 239.12 | 250.7  |

> **Highlights**
> - **100% success rate** on Stages 1, 2, and 4 (static). All environments were tested with a single model trained **only** on the Stage 4 environment. When tested on other environments, Stage 3 fails due to a **speed mismatch** between the robot and the obstacles.

### POPGym Benchmark Results

Evaluation on POPGym partially-observable tasks to validate the memory (LSTM) component in isolation from the robotics stack.

**Scan summary:** 78 runs

#### Per-Seed Breakdown — Positive Results (Best Reward, by Task)

**LSTM + PER**

| Task                 | Difficulty | Seed 42 | Seed 123 | Seed 456 | Steps (approx) |
|----------------------|:----------:|:-------:|:--------:|:--------:|:--------------:|
| RepeatPrevious       | Easy       | 1.0     | 1.0      | 1.0      | 255K           |
| PositionOnlyCartPole | Easy       | 0.4     | 0.9      | 0.6      | ~107–118K      |
| PositionOnlyCartPole | Medium     | 0.7     | 0.4      | 0.1      | ~111–122K      |
| HigherLower          | Easy       | 0.5     | 0.5      | 0.5      | 255K           |
| HigherLower          | Medium     | 0.5     | 0.5      | 0.5      | 515K           |

**LSTM + Uniform**

| Task                 | Difficulty | Seed 42 | Seed 123 | Seed 456 | Steps (approx) |
|----------------------|:----------:|:-------:|:--------:|:--------:|:--------------:|
| RepeatPrevious       | Easy       | 1.0     | 1.0      | 1.0      | 255K           |
| PositionOnlyCartPole | Easy       | 0.6     | 0.7      | 0.6      | ~107–118K      |
| PositionOnlyCartPole | Medium     | 0.6     | 0.4      | 0.5      | ~107–120K      |
| HigherLower          | Easy       | 0.5     | 0.5      | 0.5      | 255K           |

**MLP (Baseline)**

| Task                 | Difficulty | Seed 42 | Seed 123 | Seed 456 | Steps (approx) |
|----------------------|:----------:|:-------:|:--------:|:--------:|:--------------:|
| PositionOnlyCartPole | Easy       | 0.2     | 0.2      | 0.2      | ~113–119K      |
| PositionOnlyCartPole | Medium     | 0.1     | 0.1      | 0.1      | ~109–113K      |

> **Notes**
> - Best-performing task overall: **RepeatPrevious (Easy)** with a reward of **1.0** across all seeds and buffer types.

### POMuJoCo Benchmark Results

Evaluation on partially-observable MuJoCo continuous-control tasks (default comparison at 3M steps) - rewards.

**LSTM + PER**

| Task        | Seed 42 | Seed 123 | Seed 456 |
|-------------|:-------:|:--------:|:--------:|
| Ant         | 3501    | 3392     | 1822     |
| HalfCheetah | 14900   | 7900     | 12709    |

**LSTM + Uniform**

| Task        | Seed 42 | Seed 123 | Seed 456 |
|-------------|:-------:|:--------:|:--------:|
| Ant         | 1791    | 2461     | 1623     |
| HalfCheetah | 9201    | 7422     | 7921     |

---

## System Architecture

### Container Isolation

```
Container "baseline"       → Zenoh port 7507, Gazebo port 11397
Container "new_obs_space"  → Zenoh port 7485, Gazebo port 11412
Container "reward_shaping" → Zenoh port 7543, Gazebo port 11389
```

Each container is **fully isolated** automatically. When you open a terminal, `bash.bashrc`:

1. **Zenoh Router**: Generates a unique port from the workspace name (hash-based), overwrites the default Zenoh config (shared memory disabled), and starts `rmw_zenohd` in the background.
2. **Gazebo**: Assigns a unique `GAZEBO_MASTER_URI` port per workspace so multiple Gazebo instances run simultaneously.
3. **PyTorch**: Detects GPU compute capability and installs the matching CUDA wheel at build time.

**Port Assignment Logic** (deterministic — same name always yields same port):

```bash
ZENOH_PORT=$(( 7448 + $(echo "$WORKSPACE_NAME" | cksum | cut -d' ' -f1) % 100 ))
GAZEBO_PORT=$(( 11345 + $(echo "$WORKSPACE_NAME" | cksum | cut -d' ' -f1) % 100 ))
```

### Parallel Experiment Layout

```
~/workspaces/Mapless_RL_Navigation_dev/                          ← Main workspace
    └── test_turtlebot3_src/                                     ← Shared ROS workspace

~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/
    ├── baseline/          ← branch: exp/baseline
    ├── new_obs_space/     ← branch: exp/new-obs-space
    └── reward_shaping/    ← branch: exp/reward-shaping
```

All containers share `test_turtlebot3_src` via bind mount at `/opt/turtlebot3_ws`. No file duplication.

---

## Project Structure

| Directory/File | Purpose |
|---|---|
| `.devcontainer/Dockerfile` | Docker image (ROS2, Gazebo, Zenoh + Gazebo auto-isolation) |
| `.devcontainer/devcontainer.json` | Container config, mounts, extensions, env vars |
| `.scripts/post_create.sh` | Post-build setup (colcon, venv, PyTorch, requirements) |
| `.scripts/install_torch.sh` | GPU auto-detection and PyTorch installer |
| `.scripts/start_tmux.sh` | Preconfigured tmux session |
| `src/config/config.py` | Global training/environment configuration |
| `src/envs/` | Gazebo environment interfaces |
| `src/train/` | RL training scripts (SAC-LSTM) |
| `src/sac_agent/` | SAC algorithm implementation |
| `src/benchmark/` | Automated parallel benchmark framework |
| `models/` | Saved model checkpoints |
| `runs/` | TensorBoard log directories |
| `test_turtlebot3_src/` | ROS2 TurtleBot3 colcon workspace (shared across containers) |
| `requirements.txt` | Python dependencies (PyTorch handled by install_torch.sh) |

---

## Getting Started

### 1. Launch the Simulation Environment

Use the **tb3setup** window:

```bash
export GAZEBO_PLUGIN_PATH=/opt/turtlebot3_ws/build/turtlebot3_gazebo
ros2 launch turtlebot3_gazebo turtlebot3_dqn_stage4_moving_obs.launch.py gui:=false
```

### 2. Run the RL Pipeline

Use the **tb3venv** windows.

**Start the Initial Pose Node:**

```bash
python3 src/envs/gazebo_initial_pose_node_for_all_models.py
```

**Train the Model:**

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia python3 src/train/train_lstm.py --train
```

**Test a Trained Model:**

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia python3 src/train/train_lstm.py --test --model_path ./models/sac_lstm_checkpoints/best_eval_model_ep_500
```

### 3. Visualize Training Metrics (TensorBoard)

```bash
tensorboard --logdir ./ --port 6007
```

(or use the `tens` alias)

---

## Automated Benchmark System

The benchmark framework runs **multiple RL algorithm variants** in parallel, each in its own isolated devcontainer with independent Gazebo simulation.

```
~/workspaces/Mapless_RL_Navigation_dev/          ← Main workspace (source code)
~/workspaces/benchmark_worktrees/
    ├── lstm_per_seed42/                         ← Git worktree + devcontainer
    ├── lstm_uniform_seed42/                     ← Git worktree + devcontainer
    └── mlp_seed42/                              ← Git worktree + devcontainer
```

Each benchmark container gets: its own git worktree (`benchmark/<run_id>`), isolated Zenoh + Gazebo ports, a tmux session (`gazebo`, `pose`, `train`), and automatic GUI→headless Gazebo warmup with robot health checks.

### Running Benchmarks

```bash

### Dry run — show what would be executed

python -m benchmark.runner --mode parallel --dry-run

### Run all experiments

python -m benchmark.runner --mode parallel

### Run specific experiments

python -m benchmark.runner --mode parallel --experiments sac_lstm_per sac_mlp

### Run with specific seeds

python -m benchmark.runner --mode parallel --seeds 42 123 456

### Skip Gazebo warmup (if containers already have Gazebo running)

python -m benchmark.runner --mode parallel --skip-gazebo-warmup

### Override Gazebo retries and warmup time

python -m benchmark.runner --mode parallel --gazebo-retries 3 --gazebo-warmup-time 30

### Cleanup all worktrees and stop containers

python -m benchmark.runner --mode cleanup
```

### Benchmark Configuration

Edit `src/benchmark/config.py`:

```python
SEEDS = [42]                          # Seeds to test (expand: [42, 123, 456])
GAZEBO_GUI_WARMUP_SECONDS = 40        # GUI warmup time for obstacle spawning
GAZEBO_MAX_RETRIES = 5                # Max attempts to spawn robot

EXPERIMENTS = {
    "sac_lstm_per": {
        "description": "SAC + LSTM + PER (Full Pipeline)",
        "script": "train/train_lstm.py",
        "args": ["--train", "--buffer-type", "per"],
        "tag": "lstm_per",
    },
    "sac_lstm": {
        "description": "SAC + LSTM (Uniform Buffer, No PER)",
        "script": "train/train_lstm.py",
        "args": ["--train", "--buffer-type", "uniform"],
        "tag": "lstm_uniform",
    },
    "sac_mlp": {
        "description": "SAC + MLP (Feedforward Baseline)",
        "script": "train/train_mlp.py",
        "args": ["--train"],
        "tag": "mlp",
    },
}
```

### Debug Mode (Sequential, Single Container)

```bash

### Gazebo + pose node must already be running in the current container

python -m benchmark.runner --mode debug --experiments sac_lstm_per
python -m benchmark.runner --mode debug --dry-run
```

---

## Running Multiple Experiments in Parallel

This project supports running **multiple fully-isolated training instances** simultaneously using Git Worktrees.

### Setup Experiments

```bash
cd ~/workspaces/Mapless_RL_Navigation_dev

### Create experiment branches

git branch exp/baseline
git branch exp/reward-shaping
git branch exp/new-obs-space

### Create worktrees

mkdir -p ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments
git worktree add ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline exp/baseline
git worktree add ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping exp/reward-shaping
git worktree add ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space exp/new-obs-space
```

### Build All Experiment Containers

```bash
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping --build-no-cache --remove-existing-container
```

### Launch tmux in Each Container

```bash
devcontainer exec --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline bash -lc '/workspaces/baseline/.scripts/start_tmux.sh'
devcontainer exec --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space bash -lc '/workspaces/new_obs_space/.scripts/start_tmux.sh'
devcontainer exec --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping bash -lc '/workspaces/reward_shaping/.scripts/start_tmux.sh'
```

### Make Code Changes Per Experiment

Code changes on the host are **instantly reflected** inside the container (bind mount). No rebuild needed for Python changes.

```bash
cd ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping
vim src/config/config.py
git add -A && git commit -m "exp: aggressive reward shaping for dynamic obstacles"
```

### Update All Worktrees from Main

```bash
cd ~/workspaces/Mapless_RL_Navigation_dev
git pull origin main

git -C ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline rebase main
git -C ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space rebase main
git -C ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping rebase main
```

### Rebuild After Infrastructure Changes

Only needed if `Dockerfile`, `devcontainer.json`, `.scripts/`, or `requirements.txt` changed:

```bash
docker ps -aq | xargs -r docker rm -f
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping --build-no-cache --remove-existing-container
```

### Merge Experiment Back

```bash
cd ~/workspaces/Mapless_RL_Navigation_dev
git checkout main
git merge exp/reward-shaping -m "merge: reward shaping experiment - best results"
```

### Cleanup

```bash
docker stop $(docker ps -q --filter "label=devcontainer.local_folder=$HOME/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space")
git worktree remove ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space
git branch -d exp/new-obs-space
```

### Git Worktree Quick Reference

```bash
git worktree list                          # List all worktrees
git worktree add <path> <branch>           # Create new worktree
git worktree remove <path>                 # Remove a worktree
git diff main..exp/reward-shaping          # Compare experiments
git -C <path> rebase main                  # Update worktree from main
```

---

## Automatic Container Isolation

Each container is **fully isolated** automatically — no manual configuration needed. See [System Architecture](#system-architecture) for the port-assignment mechanism.

**You do NOT need to manually run `ros2 run rmw_zenoh_cpp rmw_zenohd`.** It starts automatically.

**Check the Zenoh router port on the host:**

```bash
ss -tlnp | grep rmw_zenohd
```

**Check the Gazebo port:**

```bash
ss -tlnp | grep 113
echo $GAZEBO_MASTER_URI   # inside an individual container
```

### Temporary Workaround: Gazebo Port Conflict

If the automatic `GAZEBO_MASTER_URI` is not yet configured (before Dockerfile rebuild), set unique ports manually:

```bash
export GAZEBO_MASTER_URI=http://localhost:11400   # Container 1 (baseline)
export GAZEBO_MASTER_URI=http://localhost:11401   # Container 2 (new_obs_space)
export GAZEBO_MASTER_URI=http://localhost:11402   # Container 3 (reward_shaping)
```

---

## Development Environment Setup

### 1. Build and Start the Container

First time or after changing the Dockerfile — full rebuild:

```bash
cd ~/workspaces/Mapless_RL_Navigation_dev
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

Normal start (already built):

```bash
devcontainer up --workspace-folder .
```

This builds the Docker image (ROS2 Humble, Gazebo, Python tools, tmux), starts the container with GPU passthrough and X11 forwarding, and runs `postCreateCommand` which: builds the TurtleBot3 colcon workspace, creates a Python venv, auto-detects GPU and installs the correct PyTorch build, installs pip dependencies.

### 2. Launch the tmux Session

```bash
devcontainer exec --workspace-folder . bash -lc '/workspaces/Mapless_RL_Navigation_dev/.scripts/start_tmux.sh'
```

Reattach later:

```bash
devcontainer exec --workspace-folder . tmux attach -t ros2
```

### 3. Attach VS Code (Optional)

```bash
devcontainer open --workspace-folder .
```

Or in VS Code: `Ctrl+Shift+P` → **"Dev Containers: Attach to Running Container..."**. Closing VS Code does **not** stop running processes.

---

## Prerequisites & Installation

### Prerequisites

- Docker with NVIDIA GPU support (`nvidia-container-toolkit`)
- [DevContainer CLI](https://github.com/devcontainers/cli): `npm install -g @devcontainers/cli`
- VS Code with Dev Containers extension (optional, for GUI editing/debugging)

### Installing DevContainer CLI

**Method 1: Using NVM (Recommended)**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
npm install -g @devcontainers/cli
devcontainer --version
```

**Method 2: Using NodeSource APT Repository**

```bash
sudo apt remove -y nodejs npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g @devcontainers/cli
devcontainer --version
```

> **Note:** DevContainer CLI requires Node.js >= 20. Method 1 (NVM) is preferred — no `sudo`, no interference with system packages.

---

## Troubleshooting

### Gazebo GUI Not Showing

```bash
echo $DISPLAY
export DISPLAY=:3
xclock
```

### Gazebo Fails in Multiple Containers (Exit Code 255)

```bash
echo $GAZEBO_MASTER_URI
export GAZEBO_MASTER_URI=http://localhost:11400
ros2 launch turtlebot3_gazebo turtlebot3_dqn_stage4_moving_obs.launch.py gui:=false
```

Use different port numbers (11400, 11401, 11402...) per container.

### Container Won't Start

```bash
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

### ROS2 Nodes Can't Communicate

```bash
pgrep -f rmw_zenohd          # Check if router is running
source /etc/bash.bashrc      # Trigger auto-start
ros2 topic list
```

### POSIX SHM Error (OS error 12)

Handled automatically by disabling shared memory in Zenoh config. If it appears, rebuild:

```bash
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

### PyTorch CUDA Issue (Wrong GPU Architecture)

Handled automatically by `.scripts/install_torch.sh`. On a new machine, just rebuild.

| GPU | Compute Cap | PyTorch Wheel |
|-----|-------------|---------------|
| RTX 5070/5080/5090 | sm_120 | `cu132` |
| RTX 4060-4090 | sm_89 | `cu124` |
| RTX 3060-3090 | sm_86 | `cu121` |

### tmux Session Already Exists

```bash
devcontainer exec --workspace-folder . tmux kill-session -t ros2
devcontainer exec --workspace-folder . bash -lc '/workspaces/<name>/.scripts/start_tmux.sh'
```

### Colcon Build Fails

```bash
devcontainer exec --workspace-folder . bash -lc "source /opt/ros/humble/setup.bash && cd /opt/turtlebot3_ws && rm -rf build install log && colcon build --symlink-install"
```

### ROS2 Daemon Crash (`!rclpy.ok()`)

**Symptom:** `ros2` CLI fails with `xmlrpc.client.Fault: <Fault 1: "...!rclpy.ok()">`.
**Cause:** The ROS2 daemon crashes when many nodes start rapidly during Gazebo launch.
**Fix:**

```bash
ros2 daemon stop
ros2 daemon start
sleep 2
ros2 topic list
```

> The benchmark launcher (`launcher.py`) auto-restarts the daemon before each `/scan` health check.

### Benchmark `/scan` Not Detected

**Cause:** Crashed ROS2 daemon inside the container.

```bash
docker exec -it $(docker ps -q --filter "label=devcontainer.local_folder=$HOME/workspaces/benchmark_worktrees/lstm_per_seed42") bash
source /opt/ros/humble/setup.bash
ros2 daemon stop && ros2 daemon start && sleep 2
ros2 topic list
ros2 topic echo /scan sensor_msgs/msg/LaserScan --once
```

### Cannot Display GUI (`gzclient`/`rviz2`) from Inside Container

**Symptom:** `qt.qpa.xcb: could not connect to display :0`

```bash
ls /tmp/.X11-unix/     # Check available displays
export DISPLAY=:1
gzclient
```

Alternative — allow Docker X11 access from the **host**:

```bash
xhost +local:docker
docker exec -it -e DISPLAY=$DISPLAY <CONTAINER_ID> bash
gzclient
```