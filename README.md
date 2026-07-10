# Mapless RL Navigation with TurtleBot3

This repository contains the code and simulation environments for a Mapless Reinforcement Learning navigation system using a TurtleBot3 in ROS 2 and Gazebo. The training utilizes a Soft Actor-Critic (SAC) algorithm with LSTM for continuous control in both static and dynamic scenarios.

## Prerequisites

- Docker with NVIDIA GPU support (`nvidia-container-toolkit`)
- [DevContainer CLI](https://github.com/devcontainers/cli): `npm install -g @devcontainers/cli`
- VS Code with Dev Containers extension (optional, for GUI editing/debugging)

## Installing DevContainer CLI

### Method 1: Using NVM (Recommended)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
npm install -g @devcontainers/cli
devcontainer --version
```

### Method 2: Using NodeSource APT Repository

```bash
sudo apt remove -y nodejs npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g @devcontainers/cli
devcontainer --version
```

> **Note:** The DevContainer CLI requires Node.js >= 20. Method 1 (NVM) is preferred as it doesn't require `sudo` and doesn't interfere with system packages.

## Development Environment Setup

### 1. Build and Start the Container

For the first time or after changing the Dockerfile, run a full rebuild:

```bash
cd ~/workspaces/Mapless_RL_Navigation_dev
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

For a normal start (container already built):

```bash
devcontainer up --workspace-folder .
```

This builds the Docker image (ROS2 Humble, Gazebo, Python tools, tmux), starts the container with GPU passthrough and X11 forwarding, and runs the `postCreateCommand` which:
- Builds the TurtleBot3 colcon workspace
- Creates a Python virtual environment
- Auto-detects GPU and installs the correct PyTorch build
- Installs all pip dependencies

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

Or from VS Code: `Ctrl+Shift+P` → **"Dev Containers: Attach to Running Container..."**

Closing VS Code does **not** stop any running processes.

---

## Automatic Container Isolation

Each container is **fully isolated** automatically. No manual configuration needed.

### How It Works

When you open a terminal inside the container, `bash.bashrc` automatically:

1. **Zenoh Router**: Generates a unique port from the workspace name (hash-based), overwrites the default Zenoh config files (with shared memory disabled), and starts `rmw_zenohd` in the background
2. **Gazebo**: Assigns a unique `GAZEBO_MASTER_URI` port per workspace so multiple Gazebo instances can run simultaneously
3. **PyTorch**: Detects GPU compute capability and installs the matching CUDA wheel at build time

```
Container "baseline"       → Zenoh port 7507, Gazebo port 11397
Container "new_obs_space"  → Zenoh port 7485, Gazebo port 11412
Container "reward_shaping" → Zenoh port 7543, Gazebo port 11389


```

To check gazebo port using diff port in host: 
```
ss -tlnp | grep rmw_zenohd
```
To check gazebo port using diff port in host: 
```
ss -tlnp | grep 113

echo $GAZEBO_MASTER_URI -> individual container
```



**You do NOT need to manually run `ros2 run rmw_zenoh_cpp rmw_zenohd`.** It starts automatically.

### Port Assignment Logic

```bash

### Unique port derived from workspace name (deterministic)

ZENOH_PORT=$(( 7448 + $(echo "$WORKSPACE_NAME" | cksum | cut -d' ' -f1) % 100 ))
GAZEBO_PORT=$(( 11345 + $(echo "$WORKSPACE_NAME" | cksum | cut -d' ' -f1) % 100 ))
```

Same workspace name → always same port. Different names → different ports.

### Temporary Workaround: Gazebo Port Conflict

If the automatic `GAZEBO_MASTER_URI` is not yet configured (before Dockerfile rebuild), manually set unique ports per container before launching Gazebo:

```bash

### Container 1 (baseline)

export GAZEBO_MASTER_URI=http://localhost:11400

### Container 2 (new_obs_space)

export GAZEBO_MASTER_URI=http://localhost:11401

### Container 3 (reward_shaping)

export GAZEBO_MASTER_URI=http://localhost:11402
```

---

## Getting Started

### 1. Launching the Simulation Environment

Use the **tb3setup** window:

```bash
export GAZEBO_PLUGIN_PATH=/opt/turtlebot3_ws/build/turtlebot3_gazebo
ros2 launch turtlebot3_gazebo turtlebot3_dqn_stage4_moving_obs.launch.py gui:=false
```

### 2. Running the RL Pipeline

Use the **tb3venv** windows.

#### Start the Initial Pose Node

```bash
python3 src/envs/gazebo_initial_pose_node_for_all_models.py
```

#### Train the Model

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia python3 src/train/train_lstm.py --train
```

#### Test a Trained Model

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia python3 src/train/train_lstm.py --test --model_path ./models/sac_lstm_checkpoints/best_eval_model_ep_500
```

### 3. Visualizing Training Metrics (TensorBoard)

```bash
tens
```

Or manually:

```bash
tensorboard --logdir ./ --port 6007
```

---

## Running Multiple Experiments in Parallel

This project supports running **multiple fully-isolated training instances** simultaneously using Git Worktrees. Each experiment runs in its own Docker container with independent Zenoh router and Gazebo instance.

### Architecture

```
~/workspaces/Mapless_RL_Navigation_dev/                          ← Main workspace
    └── test_turtlebot3_src/                                     ← Shared ROS workspace

~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/
    ├── baseline/          ← branch: exp/baseline
    ├── new_obs_space/     ← branch: exp/new-obs-space
    └── reward_shaping/    ← branch: exp/reward-shaping
```

All containers share `test_turtlebot3_src` via bind mount at `/opt/turtlebot3_ws`. No file duplication.

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

Code changes on the host are **instantly reflected** inside the container (bind mount). No rebuild needed for Python code changes.

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

### Rebuild All Containers After Infrastructure Changes

Only needed if `Dockerfile`, `devcontainer.json`, `.scripts/`, or `requirements.txt` changed:

```bash
docker ps -aq | xargs -r docker rm -f
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/baseline --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/new_obs_space --build-no-cache --remove-existing-container
devcontainer up --workspace-folder ~/workspaces/git_worktree/mapless_rl_navigation_dev/experiments/reward_shaping --build-no-cache --remove-existing-container
```

### Merge Winning Experiment Back

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

---

## Container Management

| Task | Command |
|------|---------|
| Start container | `devcontainer up --workspace-folder .` |
| Rebuild (with cache) | `devcontainer up --workspace-folder . --remove-existing-container` |
| Full rebuild (no cache) | `devcontainer up --workspace-folder . --remove-existing-container --build-no-cache` |
| Shell into container | `devcontainer exec --workspace-folder . bash -l` |
| Launch tmux | `devcontainer exec --workspace-folder . bash -lc '/workspaces/<name>/.scripts/start_tmux.sh'` |
| Reattach tmux | `devcontainer exec --workspace-folder . tmux attach -t ros2` |
| Kill tmux session | `devcontainer exec --workspace-folder . tmux kill-session -t ros2` |
| Open VS Code | `devcontainer open --workspace-folder .` |
| Stop container | `docker stop $(docker ps -q --filter "label=devcontainer.local_folder=$(pwd)")` |
| Stop ALL containers | `docker ps -aq \| xargs -r docker rm -f` |
| Check running containers | `docker ps` |
| GPU usage (from host) | `watch nvidia-smi` |

## tmux Key Bindings

| Keys | Action |
|------|--------|
| `Ctrl+B, 0-3` | Switch to window number |
| `Ctrl+B, D` | Detach from tmux (keeps running) |
| `Ctrl+B, C` | Create a new window |
| `Ctrl+B, N` | Next window |
| `Ctrl+B, P` | Previous window |
| `Ctrl+B, &` | Kill current window |

## When to Rebuild

| Changed File | Action Needed |
|---|---|
| Python/training code (`src/`) | No rebuild needed (bind-mounted live) |
| `requirements.txt` | Rebuild container (or `pip install -r requirements.txt` inside) |
| `.devcontainer/devcontainer.json` | Rebuild container |
| `.devcontainer/Dockerfile` | Rebuild container without cache |
| `.scripts/post_create.sh` | Rebuild container |
| `.scripts/install_torch.sh` | Rebuild container |

## Shell Aliases and Functions

| Command | Description |
|---------|-------------|
| `tb3setup` | Source ROS2 + TurtleBot3 workspace, set `TURTLEBOT3_MODEL=burger` |
| `tb3venv` | Source ROS2 + TurtleBot3 + activate Python virtual environment |
| `venv` | Activate Python virtual environment only |
| `tens` | Launch TensorBoard on port 6007 |

## Project Structure

| Directory/File | Purpose |
|---|---|
| `.devcontainer/Dockerfile` | Docker image (ROS2, Gazebo, Zenoh auto-isolation, Gazebo auto-isolation) |
| `.devcontainer/devcontainer.json` | Container config, mounts, extensions, env vars |
| `.scripts/post_create.sh` | Post-build setup (colcon, venv, PyTorch, requirements) |
| `.scripts/install_torch.sh` | GPU auto-detection and PyTorch installer |
| `.scripts/start_tmux.sh` | Preconfigured tmux session |
| `src/config/config.py` | Global training/environment configuration |
| `src/envs/` | Gazebo environment interfaces |
| `src/train/` | RL training scripts (SAC-LSTM) |
| `src/sac_agent/` | SAC algorithm implementation |
| `models/` | Saved model checkpoints |
| `runs/` | TensorBoard log directories |
| `test_turtlebot3_src/` | ROS2 TurtleBot3 colcon workspace (shared across containers) |
| `requirements.txt` | Python dependencies (no PyTorch — handled by install_torch.sh) |

---

## Troubleshooting

### Gazebo GUI Not Showing

```bash
echo $DISPLAY
export DISPLAY=:3
xclock
```

### Gazebo Fails in Multiple Containers (Exit Code 255)

Each container automatically gets a unique `GAZEBO_MASTER_URI`. If Gazebo still fails:

```bash

### Check current Gazebo port

echo $GAZEBO_MASTER_URI

### Manual workaround: set a unique port before launching

export GAZEBO_MASTER_URI=http://localhost:11400
ros2 launch turtlebot3_gazebo turtlebot3_dqn_stage4_moving_obs.launch.py gui:=false
```

Use different port numbers (11400, 11401, 11402...) for each container.

### Container Won't Start

```bash
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

### ROS2 Nodes Can't Communicate

The Zenoh router starts automatically. If `ros2 topic list` fails:

```bash

### Check if router is running

pgrep -f rmw_zenohd

### If not, source bashrc to trigger auto-start

source /etc/bash.bashrc
ros2 topic list
```

### POSIX SHM Error (OS error 12)

This is handled automatically by disabling shared memory in the Zenoh config. If you see this error, ensure your container was built with the latest Dockerfile:

```bash
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

### PyTorch CUDA Issue (Wrong GPU Architecture)

This is handled automatically by `.scripts/install_torch.sh`. If you switch machines, just rebuild:

```bash
devcontainer up --workspace-folder . --remove-existing-container --build-no-cache
```

The script auto-detects GPU compute capability:

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

### Git Worktree Quick Reference

```bash
git worktree list                          # List all worktrees
git worktree add <path> <branch>           # Create new worktree
git worktree remove <path>                 # Remove a worktree
git diff main..exp/reward-shaping          # Compare experiments
git -C <path> rebase main                  # Update worktree from main
```




## Automated Benchmark System

The benchmark framework runs **multiple RL algorithm variants** in parallel, each in its own isolated devcontainer with independent Gazebo simulation.

### Architecture

```
~/workspaces/Mapless_RL_Navigation_dev/          ← Main workspace (source code)
~/workspaces/benchmark_worktrees/
    ├── lstm_per_seed42/                         ← Git worktree + devcontainer
    ├── lstm_uniform_seed42/                     ← Git worktree + devcontainer
    └── mlp_seed42/                              ← Git worktree + devcontainer
```

Each benchmark container gets:
- Its own git worktree (branch: `benchmark/<run_id>`)
- Its own devcontainer with isolated Zenoh + Gazebo ports
- A tmux session with 3 windows: `gazebo`, `pose`, `train`
- Automatic GUI→headless Gazebo warmup with robot health checks

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

For debugging without parallel infrastructure:

```bash

### Gazebo + pose node must already be running in the current container

python -m benchmark.runner --mode debug --experiments sac_lstm_per
python -m benchmark.runner --mode debug --dry-run
```

---

## Monitoring Running Benchmarks

### Check GPU usage

```bash
watch nvidia-smi
```

### Enter a benchmark container

```bash
docker exec -it $(docker ps -q --filter "label=devcontainer.local_folder=/home/robolab-1/workspaces/benchmark_worktrees/lstm_per_seed42") bash
```

### Attach to tmux session inside container

```bash

### Once inside the container:

tmux ls
tmux attach -t lstm_per_seed42
```

| tmux Key | Action |
|----------|--------|
| `Ctrl+B` then `0` | Switch to gazebo window |
| `Ctrl+B` then `1` | Switch to pose window |
| `Ctrl+B` then `2` | Switch to train window |
| `Ctrl+B` then `n` | Next window |
| `Ctrl+B` then `p` | Previous window |
| `Ctrl+B` then `d` | **Detach** (exit safely without killing anything) |

### View tmux window output without attaching

```bash

### From host — see last 50 lines of training output

devcontainer exec --workspace-folder /home/robolab-1/workspaces/benchmark_worktrees/lstm_per_seed42 \
  bash -lc "tmux capture-pane -t lstm_per_seed42:train -p | tail -50"

### See gazebo window

devcontainer exec --workspace-folder /home/robolab-1/workspaces/benchmark_worktrees/lstm_per_seed42 \
  bash -lc "tmux capture-pane -t lstm_per_seed42:gazebo -p | tail -50"
```

### Monitor ROS2 topics (safe, read-only)

```bash

### Inside the container:

source /opt/ros/humble/setup.bash
source /opt/turtlebot3_ws/install/setup.bash
export TURTLEBOT3_MODEL=burger

ros2 topic echo /cmd_vel --once      # Check agent is sending commands
ros2 topic hz /scan                   # Verify simulation is alive (~5 Hz expected)
ros2 topic echo /odom --once          # Check robot position
```

---

## Observing Headless Simulation (Without Disturbing Training)

While training runs in headless mode (`gzserver` only), you can safely connect a visual viewer.

### Option 1: Connect `gzclient` (Gazebo GUI viewer)

```bash

### Enter the container

docker exec -it $(docker ps -q --filter "label=devcontainer.local_folder=/home/robolab-1/workspaces/benchmark_worktrees/lstm_per_seed42") bash

### Launch Gazebo GUI (connects to running gzserver as a viewer)

export DISPLAY=:1
gzclient
```

Close `gzclient` when done (click X or Ctrl+C). Training continues undisturbed.

### Option 2: Monitor via ROS2 topics only (no GUI needed)

```bash

### Inside container

ros2 topic echo /cmd_vel --once    # See current velocity commands
ros2 topic hz /scan                 # Verify simulation publishes at expected rate
ros2 topic echo /odom --field pose.pose.position --once  # Robot position
```

### Safety Summary

| Action | Safe? | Impact on Training |
|--------|-------|--------------------|
| `gzclient` (connect/disconnect) | ✅ Yes | None — pure viewer |
| `ros2 topic echo` / `ros2 topic hz` | ✅ Yes | None — read only |
| `tmux attach` / `Ctrl+B d` (detach) | ✅ Yes | None |
| `Ctrl+C` on gzclient | ✅ Yes | Only kills viewer |
| **DO NOT** `pkill gzserver` | ❌ No | Kills simulation! |
| **DO NOT** `ros2 topic pub /cmd_vel` | ❌ No | Overrides agent! |
| **DO NOT** `tmux kill-session` | ❌ No | Kills training! |



### ROS2 Daemon Crash (`!rclpy.ok()`)

**Symptom:** `ros2 topic list` or any `ros2` CLI command fails with:
```
xmlrpc.client.Fault: <Fault 1: "<class 'RuntimeError'>:!rclpy.ok()">
```

**Cause:** The ROS2 daemon (background discovery process) crashes when multiple nodes start rapidly during Gazebo launch. All `ros2` CLI commands communicate through this daemon — if it's dead, everything fails even though Gazebo and topics are actually running fine.

**Fix:**
```bash
ros2 daemon stop
ros2 daemon start
sleep 2
ros2 topic list   # Should work now
```

> **Note:** The benchmark launcher (`launcher.py`) automatically restarts the daemon before each `/scan` health check to prevent this issue.

### Benchmark `/scan` Not Detected

**Symptom:** Benchmark warmup repeatedly reports `✗ /scan NOT detected` even though Gazebo is running.

**Cause:** Crashed ROS2 daemon inside the container (see above). The `ros2 topic echo` command used for the health check fails silently.

**Verification:**
```bash

### Enter the container

docker exec -it $(docker ps -q --filter "label=devcontainer.local_folder=/home/robolab-1/workspaces/benchmark_worktrees/lstm_per_seed42") bash

### Test manually

source /opt/ros/humble/setup.bash
ros2 topic list  # If this fails with !rclpy.ok(), the daemon is dead

### Fix it

ros2 daemon stop && ros2 daemon start && sleep 2
ros2 topic list  # Should work now
ros2 topic echo /scan sensor_msgs/msg/LaserScan --once  # Should show data
```

**Permanent fix in `launcher.py`:** The `check_scan_topic()` method includes `ros2 daemon stop; ros2 daemon start; sleep 2` before each check.

### Cannot Display GUI (`gzclient`/`rviz2`) from Inside Container

**Symptom:**
```
qt.qpa.xcb: could not connect to display :0
This application failed to start because no Qt platform plugin could be initialized.
```

**Cause:** Container doesn't have X11 display access configured for the current display number.

**Fix — find the correct display:**
```bash

### Check what displays are available

ls /tmp/.X11-unix/

### Try different display numbers

export DISPLAY=:1
gzclient

### Or

export DISPLAY=:3
gzclient
```

**Alternative — allow Docker X11 access from host:**
```bash

### On the HOST (not inside container)

xhost +local:docker

### Then re-enter and try

docker exec -it -e DISPLAY=$DISPLAY <CONTAINER_ID> bash
gzclient
```

**Fallback — monitor without GUI:**
```bash
ros2 topic echo /cmd_vel --once
ros2 topic hz /scan
```