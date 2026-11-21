---
kind: configuration_system
name: PolicyIK 配置系统：YAML + Hydra + configclass 三层配置体系
slug: configuration_system
category: configuration_system
scope:
    - '**'
---

## 1. 总体方案

仓库包含两条独立但结构对称的管线，各自采用不同的配置框架：

- **硬件管线** (`hardware/`)：使用纯 Python `yaml.safe_load` 加载根级 `config.yaml`，再通过 `dataclasses` 将子段映射到 `EnvConfig` / `PoseRewardConfig` / `ReverseConfig` 等数据类。
- **仿真管线** (`simulation/`)：基于 Isaac Lab 的 `isaaclab.utils.configclass` 与 RL-Games 的 Hydra 集成（`@hydra_task_config`），通过 Python 类继承覆盖默认任务配置。
- **ROS2 bringup** (`hardware/ros2/openarm_right_bringup/config/`)：使用 ROS 2 标准的 YAML 控制器配置文件，由 `ros2_control` 在启动时读取。

没有发现 `.env`、`.toml`、环境变量注入或密钥管理服务；所有参数以明文文件形式管理。运行时覆盖仅通过 CLI 参数完成，无 feature flag 机制。

## 2. 关键文件

| 路径 | 角色 |
|---|---|
| `hardware/config.yaml` | 真实机械臂训练主配置（env / reward / curriculum / agent / her / demos / log / collect） |
| `hardware/policyik_hw/train.py` | 解析 `--config`、应用 CLI 覆盖、写入 `config.json` 快照 |
| `hardware/policyik_hw/envs/pose_env.py` | `EnvConfig` dataclass，接收 env 段并 `validate()` |
| `hardware/policyik_hw/envs/reward.py` | `PoseRewardConfig` dataclass，接收 reward 段并 `validate(agent_gamma=...)` |
| `hardware/policyik_hw/envs/workspace.py` | `ReverseConfig` dataclass，接收 curriculum 段 |
| `simulation/policyik_sim/tasks/env_cfg.py` | `PolicyIKEnvCfg` / `PolicyIKEnvCfg_PLAY`，继承 openarm_isaac_lab 基类并通过 `_configure` 重写奖励、观测、动作 |
| `simulation/policyik_sim/tasks/agents/rl_games_ppo_cfg.yaml` | RL-Games PPO 算法超参（Hydra entry point） |
| `simulation/policyik_sim/train.py` | 通过 `AppLauncher` + Hydra 加载任务配置，`dump_yaml` 输出最终 `env.yaml` / `agent.yaml` |
| `hardware/ros2/openarm_right_bringup/config/controllers.yaml` | ros2_control 控制器声明（joint_state / forward_position / forward_effort / gripper） |
| `hardware/ros2/openarm_right_bringup/config/control_gains.yaml` | 关节 PD 增益（被 controllers.yaml 引用） |

## 3. 架构与约定

### 3.1 硬件管线：YAML → dict → dataclass

`hardware/policyik_hw/train.py` 中的加载流程：

1. `argparse` 解析 `--config`（默认 `config.yaml`）、`--total-steps`、`--checkpoint-at`、`--init-from`、`--init-buffer`、`--step-offset`、`--urdf`、`--device`。
2. `resolve(args, cfg, base)` 把 CLI 覆盖应用到 `cfg['agent']` 和 `cfg['demos']`，并把相对路径拼接为绝对路径（`base = os.path.dirname(cfg_path)`）。
3. `yaml.safe_load` 后调用 `make_env(cfg)` → `EnvConfig(**cfg['env'])` → `PoseRewardConfig(**cfg['reward'])` → `ReverseConfig(**cfg['curriculum'], total_steps=...)`。
4. 每个 dataclass 都调用自身的 `validate()` 做字段校验。
5. 运行结束后把合并后的配置以 JSON 写入 `<save_dir>/<name>/config.json`，作为可复现的“黄金配置”快照。

```text
CLI args ──┐
           ├── resolve() ──► 修改 cfg dict
config.yaml ─┘
           ▼
EnvConfig(**cfg['env']).validate()
PoseRewardConfig(**cfg['reward']).validate(agent_gamma)
ReverseConfig(**cfg['curriculum']).validate()
OpenArmPoseEnv(env_cfg, rew_cfg, curr_cfg)
```

### 3.2 仿真管线：Isaac Lab configclass + Hydra

`simulation/policyik_sim/train.py` 通过 `isaaclab.app.AppLauncher` 启动 Omniverse，再交给 `isaaclab_tasks.utils.hydra.hydra_task_config(args_cli.task, args_cli.agent)` 装饰器加载 Hydra 配置。

- 任务配置类定义在 `tasks/env_cfg.py`，继承自 `openarm.tasks.manager_based.openarm_manipulation.unimanual.reach.config.OpenArmReachEnvCfg`。
- `__post_init__` 中调用 `_configure(self, start_pool, goal_pool, train=True)` 用 `PoolPoseCommandCfg`、`EventTerm`、`ObsTerm`、`RewTerm` 替换基类的命令、观测、奖励项。
- 训练/评估两个入口分别对应 `PolicyIKEnvCfg`（train=True，从 `data/train_start_pool.npz` / `data/train_goal_pool.npz` 采样）和 `PolicyIKEnvCfg_PLAY`（train=False，从 `data/source_set.npz` 采样）。RL-Games 的算法超参来自 `tasks/agents/rl_games_ppo_cfg.yaml`。
- 最终配置通过 `dump_yaml(..., "params/env.yaml")` 和 `dump_yaml(..., "params/agent.yaml")` 落盘。

### 3.3 ROS2 bringup 配置

`controllers.yaml` 是标准 ROS 2 `ros2_control` 配置，声明四个控制器：
- `joint_state_broadcaster`（类型 `joint_state_broadcaster/JointStateBroadcaster`）
- `forward_position_controller`（`forward_command_controller/ForwardCommandController`，7 个 `openarm_joint[1-7]`，interface=`position`）
- `forward_effort_controller`（同上，interface=`effort`，用于注入重力前馈）
- `gripper_controller`（`position_controllers/GripperActionController`，单关节 `openarm_finger_joint1`）

`control_gains.yaml` 提供 PD 增益，被 launch 文件引用。

## 4. 约定与约束

### 已观察到的约定

- 硬件配置按语义分组放在 `hardware/config.yaml` 的顶层键下：`env`、`reward`、`curriculum`、`agent`、`her`、`demos`、`log`、`collect`。
- `demos.file` 与 `demos.task_pool` 在 `resolve()` 中被转换为相对于 `config.yaml` 所在目录的绝对路径。
- `demos.task_pool_frac_schedule` 必须是形如 `[[0.0, v0], [f1, v1], ...]` 的分段线性调度，值域 `[0,1]`，分数严格递增且末项 < 1。
- `curriculum.schedule` 是 `[[run_fraction, sigma_deg], ...]` 形式的分段线性调度，由 `workspace.schedule_value` 插值。
- 硬件训练每次保存 checkpoint 的同时会写 `config.json`，其中包含 `env`、`reward`、`layout`、`agent`、`her`、`curriculum`、`demos`、`init_from`，作为该次运行的不可变快照。
- 仿真训练会把最终合并后的 `env.yaml` 与 `agent.yaml` 写入 `logs/rl_games/<config_name>/<timestamp>/params/`。
- 仿真任务通过继承 openarm_isaac_lab 的基类 EnvCfg，而不是从零定义，只覆盖差异部分（奖励核、观测通道、动作律）。
- 仿真日志可通过 `--track` + `--wandb-project-name` + `--wandb-entity` 启用 Weights & Biases，并在首次运行时 `wandb.config.update({"env_cfg": ..., "agent_cfg": ...})`。

### 明确强制的规则

- `--step-offset >= 0`，且当 `--step-offset > 0` 时必须同时传入 `--init-from` 与 `--init-buffer`，否则 `resolve()` 直接 `raise SystemExit`。（来源：`hardware/policyik_hw/train.py:48-51`）
- `demos.task_pool_frac_schedule` 的分数必须从 0.0 开始、严格递增、末项 < 1，且每段值 ∈ [0,1]，否则 `parse_schedule` 抛出 `SystemExit`。（来源：`hardware/policyik_hw/train.py:65-72`）
- 恢复训练时，检查点的 `layout.obs_dim` 必须与当前环境一致，且 `env` / `reward` 接口不能有差异，否则 `check_resume()` 抛出 `SystemExit`。（来源：`hardware/policyik_hw/train.py:181-189`）
- 演示数据缺失必要字段（`q_start` / `goal_p` / `goal_R`）或 demo npz 不匹配时，`load_demos` / `load_task_pool` 会 `raise SystemExit` 拒绝加载。（来源：`hardware/policyik_hw/train.py:75-81`, `298-300`）
- 多卡分布式训练时，`agent_cfg['params']['seed'] += app_launcher.global_rank`，设备名设为 `cuda:{local_rank}`，`env_cfg.sim.device` 同步更新。（来源：`simulation/policyik_sim/train.py:125-132`）
- 若开启 `--track` 但未提供 `--wandb-entity`，则 `ValueError("Weights and Biases entity must be specified for tracking.")`。（来源：`simulation/policyik_sim/train.py:224-227`）

### 未发现的机制

- 没有 `.env` 文件或 `os.environ` 注入的环境变量配置。
- 没有 feature flag / toggle 开关系统（除 `gain_schedule`、`zoom_obs`、`obs_prev_action` 等布尔配置项外）。
- 没有 secrets 管理；权重路径、URDF 路径、W&B entity 均以明文 CLI 或 YAML 传递。
- 没有 schema 验证库（Pydantic / jsonschema）；校验集中在各 dataclass 的 `validate()` 方法与 `resolve()` / `parse_schedule()` 中。