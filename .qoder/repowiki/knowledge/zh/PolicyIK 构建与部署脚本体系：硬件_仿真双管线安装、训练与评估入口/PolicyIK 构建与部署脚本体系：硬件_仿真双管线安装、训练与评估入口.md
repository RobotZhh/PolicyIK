---
kind: build_system
name: PolicyIK 构建与部署脚本体系：硬件/仿真双管线安装、训练与评估入口
slug: build_system
category: build_system
scope:
    - '**'
---

## 1. 总体方案

仓库没有 Makefile / Dockerfile / CI 配置，也没有统一的包管理清单（无 `setup.py`、`pyproject.toml`、`requirements.txt`）。构建与部署通过**每子模块独立的 Bash 安装脚本 + Python 模块入口**完成，分为两条平行管线：

- **hardware 管线**：面向真实 OpenArm v10 右臂，依赖 ROS 2 Jazzy (Ubuntu 24.04)、PyTorch 2.12、gymnasium 1.2.3。
- **simulation 管线**：面向 Isaac Lab PPO 训练，依赖 Isaac Sim 5.1.0 (`isaacsim[all,extscache]`)、IsaacLab v2.3.2、RL-Games。

所有可执行入口都是 `python3 -m <package>.<module>`，由 `scripts/*.sh` 包装为带参数校验和默认值的 CLI。不存在跨平台交叉编译或容器化流程。

## 2. 关键文件

| 路径 | 作用 |
|---|---|
| `hardware/install.sh` | 克隆 pinned 的 openarm_can / openarm_ros2 / openarm_description，`rosdep install` + `colcon build --symlink-install --packages-up-to openarm_right_bringup`，创建 `.venv` 并安装 `requirements/hardware.txt` |
| `simulation/install.sh` | 安装 PyTorch 2.7.0 + torchvision 0.22.0 (CUDA 12.8 wheel)，安装 isaacsim 5.1.0，clone pinned 版本的 IsaacLab 与 openarm_isaac_lab，pip install -e openarm，再装 `requirements/simulation.txt` |
| `requirements/hardware.txt` | hardware 管线的 pip 依赖（torch==2.12.0, gymnasium==1.2.3, urdf_parser_py, numpy）；注释说明 rclpy / sensor_msgs / std_msgs / PyKDL / PyYAML / openarm_description 应从 ROS apt 安装 |
| `requirements/simulation.txt` | simulation 管线的轻量依赖（numpy==1.26.0, scipy==1.15.3） |
| `hardware/scripts/train.sh` | 真实机械臂训练入口，调用 `policyik_hw.train --config config.yaml --name $NAME`，支持 `--init-from` / `--init-buffer` / `--step-offset` / `--total-steps` 断点续训 |
| `hardware/scripts/collect_demos.sh` | 采集演示数据，调用 `policyik_hw.collect_demos --task-file data/demo_tasks.json`，支持 `--dry-run` / `--start` / `--n` |
| `hardware/scripts/evaluate.sh` | 对三个 gap-bin 分别跑 20 episode 评估，再调用 `policyik_hw.metrics --eval-hz 8.13` 计算指标 |
| `hardware/scripts/train_watchdog.sh` | 守护进程：轮询 `buffer_step*` checkpoint，调用 `policyik_hw.watchdog_check`，异常时 `kill -INT` 训练进程并 `park` 机械臂 |
| `simulation/scripts/train.sh` | 三段式 PPO 训练（3500→8500→13000 epoch），最后用 `policyik_sim.average_checkpoints` 平均最近 20 个 checkpoint |
| `simulation/scripts/evaluate.sh` | 单尝试评估 → `corrected_goals` 修正任务 → 再次评估 → 7 个 transient seed 评估 → `policyik_sim.metrics` |
| `hardware/ros2/openarm_right_bringup/CMakeLists.txt` + `package.xml` | ROS 2 colcon 包定义，被 `install.sh` 的 `colcon build` 构建 |

## 3. 架构与约定

### 3.1 依赖版本固定策略
- 第三方 Git 仓库全部使用**精确 commit SHA** 锁定：`OPENARM_CAN_COMMIT=4063506...`、`OPENARM_ROS2_COMMIT=01b9f38...`、`OPENARM_DESCRIPTION_COMMIT=846a82d...`、`OPENARM_COMMIT=bad82e2...`、`ISAACLAB_TAG=v2.3.2`。
- pip 依赖在 `requirements/*.txt` 中 pin 主版本号（如 `torch==2.12.0`、`gymnasium==1.2.3`、`numpy==1.26.0`、`scipy==1.15.3`）。
- hardware 管线额外要求从系统 apt 安装 ROS 2 Jazzy 相关包（rclpy、sensor_msgs、std_msgs、PyKDL、PyYAML），不在 pip 中重复安装。

### 3.2 环境隔离
- hardware：`python3 -m venv --system-site-packages .venv`，并在 `.venv/bin/activate` 末尾追加 `source "$WS/install/setup.bash"`，使虚拟环境与 ROS workspace 共存。
- simulation：不创建 venv，直接通过 `PYTHONPATH="$PWD/third_party/openarm_isaac_lab${PYTHONPATH:+:$PYTHONPATH}"` 注入 openarm_isaac_lab 源码。

### 3.3 训练/评估入口模式
- 所有运行都通过 `bash scripts/<task>.sh [args]` 启动，脚本内部 `cd "$(dirname "$0")/.."` 后 `exec python3 -m policyik_hw.<module>` 或 `python3 -m policyik_sim.<module>`。
- 环境变量约定：`NAME`（hardware 训练 run 名）、`PYTHON`（可选覆盖解释器）、`WATCHDOG_NO_HOME`（跳过停臂）、`OMNI_KIT_ACCEPT_EULA=YES`、`PYTHONUNBUFFERED=1`。

### 3.4 中断恢复语义
- hardware 训练：Ctrl-C 会触发 trainer 保存 checkpoint + buffer，`train.sh` 支持 `--init-from` / `--init-buffer` / `--step-offset` / `--total-steps` 续训。
- demo 采集：已有输出 npz 会被追加写入，中断后可继续。
- 评估：按 task file 顺序运行，已完成的 episode 不会重跑。

### 3.5 安全护栏
- `train_watchdog.sh` 每 `POLL` 秒检查新产生的 `buffer_step*`（排除 `.her` 后缀），等待文件大小稳定后调用 `policyik_hw.watchdog_check`；返回码 2 表示 ABNORMAL，向训练进程发 `SIGINT`，最多重试两次，最终 `park` 机械臂。

## 4. 约定与约束（描述性）

- 每个子模块拥有独立的 `install.sh`，根目录没有顶层安装脚本；必须先分别运行 `hardware/install.sh` 与 `simulation/install.sh`。
- 所有第三方 Git 依赖通过 `git clone` + `git checkout <commit>` 拉取到 `third_party/` 或 `src/` 下，而非 pip 安装。
- hardware 管线要求 Ubuntu 24.04 + ROS 2 Jazzy + Python 3.12；simulation 管线要求 CUDA 12.8（对应 PyTorch 2.7.0 whl）与 Isaac Sim 5.1.0。
- Colcon 构建范围被限制为 `--packages-up-to openarm_right_bringup`，只构建 bringup 及其上游依赖。
- 训练产物统一落在 `<run_dir>/checkpoint`、`<run_dir>/buffer`、`<run_dir>/summary.json`；evaluation 产物落在 `outputs/eval/s$SEED/`。
- 不存在 Makefile、Dockerfile、CI YAML、GitHub Actions 或其他自动化流水线文件——本仓库的“构建系统”即上述 Bash 脚本集合。