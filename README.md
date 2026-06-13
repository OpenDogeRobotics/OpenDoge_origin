# OpenDoge

[中文版本](README_cn.md)

OpenDoge is an open-source small quadruped robot project, covering full-stack development from mechanical structure design, URDF simulation description, embedded firmware control, to reinforcement learning (RL) training and real-world deployment. The goal is to create a low-cost, reproducible, end-to-end quadruped robot development solution.

![OpenDoge](assets/mmexport1765721320399.jpg)

## Repository Structure

```
OpenDoge/
├── OpenDoge_hardware/       # Mechanical structure design & component datasheets
├── OpenDoge_description/    # URDF/Xacro robot description files (ROS/ROS2)
├── OpenDoge_firmware/       # Standalone C++ real-machine deployment & Python tooling
├── OpenDoge_train/          # Isaac Gym RL training framework (derived from HIMLoco)
├── OpenDoge_deploy/         # MuJoCo Sim2Sim verification & policy transfer
└── OpenDoge_origin/         # Main repository (this repo) — documentation & project overview
```

## Module Overview

### [OpenDoge_hardware](https://github.com/OpenDogeRobotics/OpenDoge_hardware)

Mechanical structure design and component documentation.

```
OpenDoge_hardware/
├── mechanical/
│   ├── V1.0/                           # Current release
│   │   ├── OpenDog/                    # 405 SolidWorks part files (.SLDPRT)
│   │   ├── 全零件.STEP                  # Full part-level assembly (~80 MB)
│   │   └── 全子装配.STEP                # Sub-assembly STEP (~82 MB)
│   └── V2.0/                           # Next iteration (in development)
└── datasheets/                         # Component specifications & reference materials
    ├── dm-imu/                         # DM-IMU-L1 6-axis IMU datasheet, firmware, examples
    ├── chiwu/                          # Motor driver board schematic & demo code
    └── EL05使用说明书2600428.pdf         # EL05 motor manual
```

**Design highlights:**
- 12-DOF quadruped configuration: 4 legs × 3 actuated joints (hip / thigh / calf)
- All mechanical parts designed in SolidWorks, STEP files compatible with Fusion 360 / FreeCAD
- Structural components suitable for 3D printing (PLA/PETG/Nylon) and CNC aluminum machining
- Foot tips recommended in rubber or TPU for improved traction

### [OpenDoge_description](https://github.com/OpenDogeRobotics/OpenDoge_description)

ROS/ROS2 robot description package providing kinematic & dynamic models and visualization meshes.

```
OpenDoge_description/
└── URDF/
    ├── urdf/
    │   ├── Opendoge.urdf              # Full URDF kinematic & dynamic model
    │   └── Opendoge.csv               # Joint/link mapping table
    ├── meshes/                        # STL mesh per link (base_link + 4 legs × 4 segments)
    │   ├── base_link.STL
    │   ├── FL_{hip,thigh,calf,foot}.STL   # Front-left leg
    │   ├── FR_{hip,thigh,calf,foot}.STL   # Front-right leg
    │   ├── RL_{hip,thigh,calf,foot}.STL   # Rear-left leg
    │   └── RR_{hip,thigh,calf,foot}.STL   # Rear-right leg
    ├── config/joint_names_Opendoge.yaml
    ├── launch/
    │   ├── display.launch             # RViz visualization
    │   └── gazebo.launch              # Gazebo simulation
    ├── xml/
    │   ├── Opendoge.xml               # MJCF model for MuJoCo
    │   └── scene.xml
    ├── CMakeLists.txt
    └── package.xml
```

**Joint mapping (12 actuated joints):**

| Leg | Hip | Thigh | Calf | Foot (fixed) |
|-----|-----|-------|------|--------------|
| FL (Front-Left) | FL_hip_joint | FL_thigh_joint | FL_calf_joint | FL_foot |
| FR (Front-Right) | FR_hip_joint | FR_thigh_joint | FR_calf_joint | FR_foot |
| RL (Rear-Left) | RL_hip_joint | RL_thigh_joint | RL_calf_joint | RL_foot |
| RR (Rear-Right) | RR_hip_joint | RR_thigh_joint | RR_calf_joint | RR_foot |

**Dependencies:** ROS/ROS2, `robot_state_publisher`, `joint_state_publisher`, `rviz`, `gazebo`.  
**License:** BSD.

### [OpenDoge_firmware](https://github.com/OpenDogeRobotics/OpenDoge_firmware)

Standalone C++ real-machine deployment with Python tooling. **No ROS dependency at runtime** — the main deployment binary `opendoge_deploy` directly manages CPU ONNX inference, 4-channel SocketCAN, EL05 motor control frames, and safety damping.

```
OpenDoge_firmware/
├── src/
│   └── opendoge_deploy/               # Non-ROS real-machine deployment (C++)
│       ├── src/
│       │   ├── main.cpp               # Entry point: state machine, control loop
│       │   ├── el05_socketcan.cpp      # EL05/RobStride CAN protocol driver
│       │   ├── onnx_policy.cpp         # ONNX Runtime policy inference backend
│       │   ├── policy.cpp              # Policy backend abstraction (ONNX / linear_csv / none)
│       │   └── runtime_io.cpp         # Command file & IMU file I/O
│       └── include/opendoge_deploy/    # Headers (types, policy, CAN, I/O)
├── scripts/
│   ├── setup_can.sh                   # Initialize 4× SocketCAN interfaces
│   ├── setup_vcan.sh                  # Virtual CAN for hardware-less testing
│   ├── setup_onnx.sh                  # Download & install ONNX Runtime locally
│   └── start_robot.sh                 # One-shot launch: CAN + IMU + joystick + deploy
├── tools/
│   ├── el05/                          # Interactive EL05 motor test menu & protocol self-test
│   ├── imu/dm_imu_bridge.py           # DM-IMU-L1 serial → state file bridge
│   ├── joystick/                      # Xbox gamepad → command file bridge
│   └── usb2can/                       # USB2CAN example & motor demo scripts
├── policy/                            # ONNX policy models for deployment
├── Changelog/Codex.md                 # Development changelog
├── docs/                              # Reference manuals
└── requirements.txt
```

**Deployment pipeline:**

```text
opendoge_deploy (single process)
  → CPU ONNX policy inference @ 100 Hz
  → Position target hold @ 200 Hz
  → EL05 q/dq/tau/kp/kd control @ 1000 Hz
  → SocketCAN can0/can1/can2/can3
  → USB-to-4×CAN adapter
  → 12× EL05/RobStride motors
```

**Motor-to-CAN mapping:** each CAN bus drives one leg (3 motors):

```text
can0: FL  (motors 1/2/3  = hip/thigh/calf)
can1: FR  (motors 4/5/6  = hip/thigh/calf)
can2: RL  (motors 7/8/9  = hip/thigh/calf)
can3: RR  (motors 10/11/12 = hip/thigh/calf)
```

**Key features:**
- **Multiple policy backends:** ONNX Runtime (CPU), linear CSV, or none (damping-only)
- **Safety-first design:** automatic damping mode on motor fault, over-temperature, feedback timeout, CAN error, or E-Stop trigger
- **Realtime support:** optional `SCHED_FIFO` + `mlockall` for low-latency control
- **Flexible I/O:** command input via file (compatible with joystick bridge), IMU input via serial bridge — no ROS topics required
- **Dry-run mode:** full control loop validation without hardware (vCAN or no-CAN modes)

### [OpenDoge_train](https://github.com/OpenDogeRobotics/OpenDoge_train)

Isaac Gym-based RL training framework derived from [HIMLoco](https://github.com/InternRobotics/HIMLoco).

```
OpenDoge_train/
├── legged_gym/                        # Core training framework
│   ├── envs/
│   │   ├── base/                      # Base classes (LeggedRobot, LeggedRobotCfg)
│   │   ├── opendoge/                  # OpenDoge training config & task registration
│   │   ├── a1/ go1/                   # Unitree A1 / Go1 configs
│   │   └── go2/ g1/ h1/ h1_2/ zsl1/   # Additional robot configs
│   ├── scripts/
│   │   ├── train.py                   # Training entry point
│   │   └── play.py                    # Playback & ONNX export
│   ├── utils/                         # Task registry, logger, terrain, math helpers
│   └── sim2sim.py                     # MuJoCo Sim2Sim validation entry
├── rsl_rl/                            # HIMLoco PPO implementation (forked from RSL-RL)
│   └── rsl_rl/
│       ├── algorithms/                # HIMPPO, PPO
│       ├── modules/                   # HIMActorCritic (LSTM), HIMEstimator
│       ├── runners/                   # HIMOnPolicyRunner
│       └── storage/                   # HIMRolloutStorage
├── deploy/
│   ├── deploy_mujoco/                 # MuJoCo Sim2Sim scripts & configs
│   ├── deploy_real/                   # Real-robot deployment (Unitree Go2/G1/H1)
│   └── pre_train/                     # Pre-trained model weights
├── resources/robots/Opendoge/         # OpenDoge URDF + MuJoCo XML + STL meshes
├── Tool/
│   ├── check_urdf.py                  # URDF validation utility
│   └── simplify_mesh.py               # Mesh decimation utility
├── tools/
│   └── mujoco_ik/                     # IK gait controller & reference motion recorder
│       ├── opendoge_mujoco/            # IK solver, IMU feedback, gait planner
│       ├── scripts/                    # PD control & keyboard IK demos
│       └── configs/
├── scripts/export_onnx.py             # Standalone ONNX export
├── onnx/                              # Exported ONNX policy models
├── docs/                              # Training tuning records
├── setup.py
├── himloco.yml                        # Conda environment specification
└── setup_env.sh                       # Environment setup helper
```

**Quick start (training):**
```bash
# Environment setup
conda create -n himloco python=3.8
conda activate himloco
pip install torch==2.3.1 torchvision==0.18.1 --index-url https://download.pytorch.org/whl/cu121
pip install mujoco==3.2.3
# Install Isaac Gym Preview 4 from NVIDIA
pip install -e <isaacgym_path>/isaacgym/python
pip install -e .

# Train OpenDoge
export PYTHONPATH=$PWD
python legged_gym/scripts/train.py --task=opendoge --headless --num_envs=4096

# Resume from checkpoint
python legged_gym/scripts/train.py --task=opendoge --resume --load_run <run_name> --checkpoint <N>
```

**Playback & export:**
```bash
python legged_gym/scripts/play.py --task=opendoge --load_run <run_name>
# ONNX model saved to onnx/ directory
```

**Key features:**
- **Curriculum learning:** progressive training from standing to dynamic locomotion
- **Domain randomization:** friction, motor strength, payload, external pushes, PD randomization
- **RNN policy:** LSTM-based Actor-Critic for memory-dependent gait patterns
- **Multi-robot support:** OpenDoge, Unitree A1/Go1/Go2, H1/H1_2, G1, ZSL1
- **Sim2Sim export:** MuJoCo XML + ONNX policy for deployment verification
- **TensorBoard monitoring:** full scalar logging for all reward components

### [OpenDoge_deploy](https://github.com/OpenDogeRobotics/OpenDoge_deploy)

MuJoCo-based Sim2Sim verification and policy deployment validation.

```
OpenDoge_deploy/
├── deploy/
│   └── deploy_mujoco/
│       ├── deploy_opendoge.py          # Keyboard-controlled Sim2Sim
│       ├── deploy_opendoge_xbox.py     # Xbox gamepad-controlled Sim2Sim
│       ├── onnx_path_utils.py          # ONNX model path resolution
│       └── configs/opendoge.yaml       # Deployment config (PD gains, observation scaling)
├── onnx/                               # ONNX policy models
│   ├── flat_opendoge_9000_omni.onnx    # Omnidirectional policy, 9000 iterations (recommended)
│   ├── flat_opendoge_fresh_6000.onnx   # Fresh policy, 6000 iterations
│   ├── flat_opendoge_5700.onnx         # Gen4-style policy, 5700 iterations
│   └── flat_opendoge_gen52_4800.onnx   # Gen52 policy, 4800 iterations
└── resources/robots/Opendoge/          # MJCF model + STL meshes
```

**Quick start:**
```bash
conda activate himloco
# Keyboard control
python deploy/deploy_mujoco/deploy_opendoge.py
# Xbox gamepad control
python deploy/deploy_mujoco/deploy_opendoge_xbox.py
# Specify ONNX model
python deploy/deploy_mujoco/deploy_opendoge.py --onnx onnx/flat_opendoge_9000_omni.onnx
```

**PD parameter alignment (deploy ↔ train):**

| Parameter | Deploy Value | Train Value |
|-----------|-------------|-------------|
| kp | 12.0 | 12.0 |
| kd | 0.5 | 0.5 |
| action_scale | 0.30 | 0.30 |
| control_decimation | 2 (100 Hz) | 2 (100 Hz) |
| simulation_dt | 0.005 (200 Hz) | 0.005 (200 Hz) |
| init_base_height | 0.15 m | 0.15 m |

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Mechanical Design** | SolidWorks | Part design, assembly STEP export |
| **Simulation Model** | URDF / Xacro / MJCF | Kinematic & dynamic description |
| **Physics Simulation** | Isaac Gym + MuJoCo + Gazebo | Isaac Gym for RL training, MuJoCo for Sim2Sim, Gazebo for ROS simulation |
| **Robot Middleware** | ROS2 Humble (optional) | Communication & control (description package only) |
| **Hardware Control** | SocketCAN + EL05 protocol | Real-time motor control via CAN bus |
| **Sensors** | DM-IMU-L1 | 6-axis attitude sensor |
| **DL Framework** | PyTorch 2.3.1 + CUDA 12.1 | RL model training |
| **RL Algorithm** | HIMLoco (PPO) | LSTM Actor-Critic architecture |
| **Model Export** | ONNX Runtime 1.18+ | PyTorch → ONNX conversion & CPU inference |
| **Monitoring** | TensorBoard | Training metrics visualization |
| **Deployment** | Standalone C++ binary | Real-time control loop on RK3588 / x86 |
| **Language** | Python (train/sim), C++ (deploy) | |

## Key Dependencies

- CUDA 12.1, PyTorch 2.3.1 GPU, Isaac Gym Preview 4
- MuJoCo 3.2.3, ONNX Runtime ≥ 1.18
- ROS2 Humble + colcon (description & tools only; deployment is ROS-free)

## Quick Start

### Hardware Manufacturing
1. Use STEP files in `OpenDoge_hardware/mechanical/V1.0/` for 3D printing or CNC machining
2. Assemble the robot body following the sub-assembly hierarchy

### Simulation Training
```bash
cd OpenDoge_train
conda env create -f himloco.yml
conda activate himloco
pip install -e .
python legged_gym/scripts/train.py --task=opendoge --headless --num_envs=4096
```

### Sim2Sim Verification
```bash
cd OpenDoge_deploy
conda activate himloco
python deploy/deploy_mujoco/deploy_opendoge.py --onnx onnx/flat_opendoge_9000_omni.onnx
```

### Real Robot Deployment
```bash
cd OpenDoge_firmware
# Install ONNX Runtime
./scripts/setup_onnx.sh
export ONNXRUNTIME_ROOT=$(realpath build/deps/onnxruntime)

# Build
colcon build --symlink-install --packages-select opendoge_deploy
source install/setup.bash

# Dry-run validation
./scripts/start_robot.sh dry

# Run with ONNX policy
POLICY_PATH=policy/gen52_model4800.onnx ./scripts/start_robot.sh policy
```

## Development Roadmap

- [x] **Hardware:** 405-part mechanical design, 12-DOF quadruped, 3D-printable
- [x] **URDF Model:** Full kinematic/dynamic model with STL meshes, RViz & Gazebo support
- [x] **Firmware:** Standalone C++ deployment with ONNX inference, SocketCAN, safety damping
- [x] **Training Pipeline:** Isaac Gym PPO training with curriculum learning & domain randomization
- [x] **Sim2Sim:** MuJoCo policy verification with keyboard/gamepad control
- [ ] **OpenDoge-specific training config:** Dedicated task tuned for the actual hardware
- [ ] **Real-machine walking:** First stable locomotion on physical robot
- [ ] **Gait library:** Multiple gaits (trot, walk, bound, pace) with switching
- [ ] **Terrain adaptation:** Stairs, slopes, uneven ground
- [ ] **Vision integration:** Depth camera + elevation-map-based locomotion
- [ ] **Community release v1.0:** Full BOM, build guide, pre-trained models

## Acknowledgments

This project benefits from the inspiration and support of the following open-source work:
- [HIMLoco](https://github.com/InternRobotics/HIMLoco) — Simulation training framework foundation
- [galileo-isaacgym](https://github.com/Hahalim2022y/galileo_isaacgym) — Training framework reference
- [quadruped_rl](https://github.com/Benxiaogu/quadruped_rl) — Quadruped RL reference implementation
- The ROS2, Isaac Gym, and MuJoCo open-source communities

## License

Each submodule follows its own license terms. See the `LICENSE` file in each subdirectory for details.
