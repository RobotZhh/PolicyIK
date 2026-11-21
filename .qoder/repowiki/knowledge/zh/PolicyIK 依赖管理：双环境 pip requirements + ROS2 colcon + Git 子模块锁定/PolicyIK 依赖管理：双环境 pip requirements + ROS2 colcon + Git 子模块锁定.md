---
kind: dependency_management
name: PolicyIK 依赖管理：双环境 pip requirements + ROS2 colcon + Git 子模块锁定
slug: dependency_management
category: dependency_management
scope:
    - '**'
---

## 1. 使用的系统/方法

本项目采用 **多工具、按环境拆分** 的依赖管理策略，没有统一的顶层 `pyproject.toml` / `Pipfile` / `poetry.lock`：

- **Python 包**：通过 `requirements/*.txt`（pip 格式）声明，由各自环境的 `install.sh` 安装到独立虚拟环境。
- **ROS 2 依赖**：通过 `hardware/ros2/openarm_right_bringup/package.xml` + `rosdep` + `colcon build` 管理，属于 ROS workspace 体系。
- **第三方 Python 库（非 PyPI）**：以 **Git clone + 固定 commit** 的方式放入 `third_party/`，并在安装脚本中 `checkout` 指定提交。
- **CUDA/cu128 专用 wheel**：PyTorch 与 Isaac Sim 通过自定义 `--index-url` / `--extra-index-url` 从 NVIDIA 官方源拉取。

## 2. 关键文件

- `requirements/hardware.txt` — 硬件管线 Python 依赖（torch==2.12.0, gymnasium==1.2.3, urdf_parser_py, numpy）。
- `requirements/simulation.txt` — 仿真管线 Python 依赖（numpy==1.26.0, scipy==1.15.3）。
- `hardware/install.sh` — 硬件环境安装入口：克隆 openarm_can / openarm_ros2 / openarm_description 三个仓库并 `colcon build`，然后创建 `.venv` 并 `pip install -r ../requirements/hardware.txt`。
- `simulation/install.sh` — 仿真环境安装入口：安装 torch==2.7.0 + torchvision==0.22.0（cu128）、isaacsim==5.1.0（NVIDIA PyPI），clone IsaacLab v2.3.2 与 openarm_isaac_lab（固定 commit），再 `pip install -e` 本地包并安装 `requirements/simulation.txt`。
- `hardware/ros2/openarm_right_bringup/package.xml` — ROS 2 包的依赖声明（被 `rosdep install` 消费）。

## 3. 架构与约定

### 3.1 环境隔离
两个运行环境完全独立：
- **hardware**：基于 Ubuntu 24.04 + ROS 2 Jazzy + Python 3.12。ROS 包通过 apt (`ros-jazzy-ros-base`, `python3-pykdl`, `python3-yaml`) 安装；其余 Python 包进入 `hardware/.venv`。
- **simulation**：基于 Isaac Lab v2.3.2 + Isaac Sim 5.1.0 + CUDA 12.8。依赖直接安装在当前 Python 环境中（无显式 venv 创建）。

### 3.2 版本锁定方式
| 来源 | 锁定手段 | 示例 |
|---|---|---|
| PyPI | `requirements.txt` 中的 `==` 精确版本 | `torch==2.12.0`, `gymnasium==1.2.3`, `numpy==1.26.0`, `scipy==1.15.3` |
| NVIDIA PyPI | `--extra-index-url https://pypi.nvidia.com` | `isaacsim[all,extscache]==5.1.0` |
| CUDA wheel 索引 | `--index-url https://download.pytorch.org/whl/cu128` | `torch==2.7.0 torchvision==0.22.0` |
| Git 仓库 | 变量 + `git checkout <commit>` | `OPENARM_CAN_COMMIT=4063506...`, `ISAACLAB_TAG=v2.3.2` |
| ROS 依赖 | `package.xml` + `rosdep` | `openarm_description`, `openarm_hardware` 等 |

### 3.3 第三方代码纳入策略
- `third_party/IsaacLab`：`git clone --depth 1 --branch $ISAACLAB_TAG`，仅浅克隆分支标签。
- `third_party/openarm_isaac_lab`：完整 clone 后 `git checkout $OPENARM_COMMIT` 固定提交。
- `hardware/src/openarm_*`：在 `hardware/install.sh` 中克隆 `openarm_can`、`openarm_ros2`、`openarm_description` 三个 enactic 仓库，每个都 `checkout` 固定 commit。

### 3.4 可复现性设计
- 所有外部 Git 依赖均通过**固定 commit SHA** 或**带 hash 的分支引用**锁定，避免上游漂移影响实验结果。
- `hardware/install.sh` 将 `source "$WS/install/setup.bash"` 注入 `.venv/bin/activate`，确保激活虚拟环境时 ROS 2 workspace 路径自动生效。

## 4. 约定与约束

- **Python 依赖只通过 `requirements/*.txt` 声明**，项目内未使用 `pyproject.toml`、`setup.py`、`Pipfile` 或 `poetry` 作为依赖清单（`grep_code` 搜索结果为空）。
- **硬件与仿真依赖严格分离**：`hardware.txt` 与 `simulation.txt` 分别对应两条管线，不混用；二者对 `numpy` 的版本也不同（hardware 未 pin，simulation pin 1.26.0）。
- **PyTorch 版本因环境而异**：hardware 使用 `torch==2.12.0`（来自 PyPI），simulation 使用 `torch==2.7.0`（来自 NVIDIA cu128 专用索引），这是由 `install.sh` 硬编码决定的。
- **Isaac Sim 必须从 NVIDIA 私有 PyPI 安装**：`simulation/install.sh` 显式传入 `--extra-index-url https://pypi.nvidia.com`，否则无法获取 `isaacsim`。
- **ROS 2 依赖通过 `rosdep` 而非 pip 安装**：`hardware/install.sh` 调用 `rosdep install --ignore-src -y --from-paths src/PolicyIK/hardware/ros2 src/openarm_ros2/openarm_hardware src/openarm_can src/openarm_description`，apt 包名见 `hardware.txt` 注释。
- **第三方源码一律放在 `third_party/` 或 `src/` 下并通过 `git checkout` 固定提交**，不允许 `pip install git+https://...` 形式的动态拉取。
- **`hardware/install.sh` 要求预先存在 `/opt/ros/jazzy/setup.bash`**（脚本第 9 行 `source /opt/ros/jazzy/setup.bash`），即宿主系统需预装 ROS 2 Jazzy。
- **`.venv` 通过 `--system-site-packages` 创建**，以便虚拟环境能访问 ROS 2 提供的 Python 包（如 `rclpy`、`sensor_msgs`、`std_msgs`）。

## 5. 缺失项（观察到的空白）

- 没有 `requirements.txt` 之外的锁文件（如 `requirements-hardware.lock` / `pip-tools` / `uv.lock`），因此 `pip install -r` 的行为取决于 pip 默认解析策略，并非完全确定性。
- 没有 `pyproject.toml` / `setup.py`，Python 包不是通过 `pip install .` 安装，而是通过 `install.sh` 直接 `pip install -e third_party/...` 或 `pip install -r requirements/*.txt`。
- 没有私有 PyPI 镜像或 `PIP_INDEX_URL` 全局配置，所有自定义索引均在 `install.sh` 中以命令行参数形式硬编码。
- 未见 `conda`、`mamba`、`uv`、`poetry` 等其他包管理器痕迹。