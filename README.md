# OpenDoge

OpenDoge 是一款开源的小型四足机器人项目，涵盖从机械结构设计、URDF 仿真描述、嵌入式固件控制到强化学习训练的全栈开发。

![alt text](mmexport1765721320399.jpg)

## 仓库结构

```
OpenDoge/
├── OpenDoge_hardware/       # 机械结构设计
├── OpenDoge_description/    # URDF/Xacro 机器人描述文件
├── OpenDoge_firmware/       # ROS2 运控固件工作区
├── OpenDoge_train/          # 强化学习训练框架
├── OpenDoge_deploy/         # 部署与策略迁移
└── OpenDoge_origin/         # 主仓库（本仓库）
```

## 各模块说明

### [OpenDoge_hardware](OpenDoge_hardware/)
机械结构设计文件，包含：
- 整机装配体 STEP 文件（`全零件.STEP`、`全子装配.STEP`）
- 405 个 SolidWorks 零件文件（`OpenDog/` 目录下）
- 适用于 3D 打印与 CNC 加工的结构件设计

### [OpenDoge_description](OpenDoge_description/)
ROS2 机器人描述包，包含：
- URDF 格式的机器人运动学与动力学模型（`Opendoge.urdf`）
- 各连杆与关节的 STL 网格文件（`meshes/` 目录）
- 关节/连杆映射表（`Opendoge.csv`）
- 可用于 Isaac Gym、Mujoco 等仿真器的模型导入

### [OpenDoge_firmware](OpenDoge_firmware/)
基于 ROS2 Humble 的上位机运控工作区，包含：
- `motor_control`：CAN 总线电机驱动接口
- `dm_imu`：IMU 传感器驱动
- `opendoge_description`：URDF/Xacro 模型
- `opendoge_control`：ros2_control 硬件接口配置
- `opendoge_bringup`：系统启动与控制器管理

支持电机力矩/位置/速度控制，IMU 数据采集，与强化学习策略的实时推理部署。

### [OpenDoge_train](OpenDoge_train/)
基于 Isaac Gym 的强化学习训练框架，衍生自 [HIMLoco](https://github.com/InternRobotics/HIMLoco) 与 [galileo-isaacgym](https://github.com/Hahalim2022y/galileo_isaacgym)，包含：
- **训练（Train）**：支持多种机器人配置的并行训练，可调节环境数量、随机种子、迭代次数等
- **演示（Play）**：加载训练好的模型进行仿真演示，支持导出 ONNX 策略网络
- **Sim2Sim**：Mujoco 环境下的策略迁移验证，支持键盘/手柄遥控
- **Sim2Real**：通过 LCM 通信协议将策略部署到真实机器人
- **Dashboard**：基于 Web 的训练监控与策略部署面板

### [OpenDoge_deploy](OpenDoge_deploy/)
策略部署与迁移仓库，负责将训练好的强化学习策略部署到真实机器人上，包括：
- ONNX 模型推理与运行时管理
- Sim2Real 策略迁移与适配
- 实机部署配置与工具链

## 快速开始

### 硬件制造
1. 使用 `OpenDoge_hardware/` 中的 STEP 文件进行 3D 打印或 CNC 加工
2. 按装配体文件组装机器人本体

### 仿真训练
```bash
cd OpenDoge_train
conda env create -f OpenDoge_env.yml
conda activate OpenDoge_env
python legged_gym/scripts/train.py --task=galileo --num_envs=256 --rl_device=cuda:0
```

### 实机部署
```bash
cd OpenDoge_firmware
colcon build --symlink-install
ros2 launch opendoge_bringup bringup.launch.py
```

## 致谢

本项目得益于以下开源工作的启发与支持：
- [HIMLoco](https://github.com/InternRobotics/HIMLoco) — 仿真训练框架基础
- [galileo-isaacgym](https://github.com/Hahalim2022y/galileo_isaacgym) — 训练框架参考
- [quadruped_rl](https://github.com/Benxiaogu/quadruped_rl) — 四足 RL 参考实现
- ROS2 与 Isaac Gym 开源社区

## 许可证

各子模块分别遵循其自身的许可证条款，详见各子目录中的 `LICENSE` 文件。
