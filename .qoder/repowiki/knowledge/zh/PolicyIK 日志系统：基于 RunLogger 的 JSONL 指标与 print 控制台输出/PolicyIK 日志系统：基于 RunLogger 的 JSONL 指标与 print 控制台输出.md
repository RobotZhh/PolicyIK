---
kind: logging_system
name: PolicyIK 日志系统：基于 RunLogger 的 JSONL 指标与 print 控制台输出
slug: logging_system
category: logging_system
scope:
    - '**'
---

## 1. 使用的系统与框架

仓库没有引入第三方日志库（如 `logging`、`loguru`、`tensorboard`）。所有日志输出由代码内自实现组件完成，分为两条路径：

- **硬件管线** (`hardware/policyik_hw/train.py`)：通过自定义 `RunLogger` + `_Tee` 将结构化指标写入 JSONL 文件，同时把 `sys.stdout`/`sys.stderr` 重定向到 `console.log`。
- **仿真管线** (`simulation/policyik_sim/train.py`)：仅使用 `print` 打印训练状态和 TensorBoard 路径，依赖外部 RL-Games/TensorBoard 基础设施。

`hardware/flash_rl/`（FlashSAC 算法实现）中未发现任何 `import logging` 或 logger 初始化，算法内部不产生独立日志。

## 2. 关键文件

- `hardware/policyik_hw/train.py`：定义 `RunLogger`、`_Tee`、`log_row`、`print_row`，是结构化日志的核心。
- `hardware/policyik_hw/config.yaml`：通过 `cfg["log"]` 提供 `save_dir`、`window`、`every` 等日志参数。
- `simulation/policyik_sim/train.py`：仿真侧的 `print("[INFO] Logging experiment in directory: ...")` 入口。
- `hardware/policyik_hw/collect_demos.py`：演示采集脚本，全部使用 `print` 输出进度。

## 3. 架构与约定

### 3.1 RunLogger（硬件训练）

```python
class RunLogger:
    def __init__(self, out_dir):
        os.makedirs(os.path.join(out_dir, "checkpoint"), exist_ok=True)
        self._m = open(os.path.join(out_dir, "metrics.jsonl"), "w")
        self._e = open(os.path.join(out_dir, "episodes.jsonl"), "w")
        self._c = open(os.path.join(out_dir, "console.log"), "a", buffering=1)
        self._orig = (sys.stdout, sys.stderr)
        sys.stdout = _Tee(sys.stdout, self._c)
        sys.stderr = _Tee(sys.stderr, self._c)
```

- 每个运行目录 (`out_dir`) 固定产出三个文件：`metrics.jsonl`、`episodes.jsonl`、`console.log`。
- `_Tee` 是一个极简的 `io` 适配器，同时写终端句柄和 `console.log`，保留原始 stdout/stderr 行为（通过 `__getattr__` 透传）。
- `metric(row)` 与 `episode(row)` 均调用 `json.dumps(row) + "\n"` 并立即 `flush()`，保证每条记录落盘。
- `close()` 恢复原始 stdout/stderr 后关闭文件。

### 3.2 指标行结构 (`log_row`)

`log_row` 构造的 dict 字段是硬编码的，构成 `metrics.jsonl` 的行 schema：

| 字段 | 含义 |
|---|---|
| `step` / `wall_s` / `n_episodes` / `new_episodes` | 基础计数 |
| `r`, `l`, `final_dist`, `min_dist`, `final_ang_deg` | 最近 `cfg["log"]["window"]` 条 episode 的均值 |
| `settled_success`, `strict_settled` | 成功率（均值） |
| `term_means` | 各 reward term 的均值映射 |
| `actor/loss`, `critic/loss`, `actor/entropy`, `temperature/value` | FlashSAC 内部信息（来自 `hooks.last_info`） |
| `replay`, `global_step`, `run_frac`, `pool_frac_target`, `steps_per_s` | 回放缓冲与调度元数据 |
| `step_wall_s_mean`, `upd_step_s_per_step`, `n_upd_step_per_step`, `upd_reset_s`, `n_upd_reset`, `est_update_s` | 计时统计 |
| `pool_frac` | 当前任务池采样比例均值 |

Episode 行 (`logger.episode(s)`) 直接透传 `info["episode"]`，追加 `step`、`wall_s`、`terminated`、`from_pool`、`sigma_after` 以及 curriculum 事件信息。

### 3.3 控制台输出格式

`print_row` 以 `[step/total gN]` 前缀打印人类可读的一行摘要，包含回报、距离、角度、成功率、critic loss、温度、timing 与 pool 比例。所有数值经 `_fmt(x, ".3f")` 格式化，非有限值显示为 `n/a`。

### 3.4 仿真侧日志

`simulation/policyik_sim/train.py` 仅用 `print("[INFO] Logging experiment in directory: {log_root_path}")` 告知 TensorBoard 输出位置；实际指标由 RL-Games 的 `VecEnvWrapper` 与 TensorBoard 回调处理，本仓库未封装该部分。

## 4. 约定与约束

- **无 log level**：仓库不使用 Python `logging` 模块，不存在 DEBUG/INFO/WARNING/ERROR 分级；所有输出均为同级别文本。
- **JSONL 作为唯一结构化持久化格式**：`RunLogger.metric` 与 `RunLogger.episode` 都使用 `json.dumps(row)` 逐行写入，且每次写入后立即 `flush()`，确保崩溃可恢复。
- **输出目录约定**：`out_dir` 下固定存在 `checkpoint/`、`metrics.jsonl`、`episodes.jsonl`、`console.log`、`config.json`、`summary.json`；这是 `main()` 与 `write_summary` 共同约定的产物清单。
- **日志频率受配置驱动**：`cfg["log"]["every"]` 控制 metric 行写入间隔，`cfg["log"]["window"]` 控制滚动均值窗口大小（见 `log_row` 中 `recent = eps[-cfg["log"]["window"]:]`）。
- **stdout/stderr 全局重定向**：`RunLogger.__init__` 在构造时替换 `sys.stdout`/`sys.stderr`，因此 `print(...)` 也会进入 `console.log`；`close()` 负责恢复原始流。
- **中断安全**：`main()` 注册 SIGINT 处理器，第二次 Ctrl-C 才会真正退出；第一次会设置 `state["stop"]` 并在循环末尾触发保存与 `logger.close()`。
- **仿真侧无统一 logger**：`simulation/policyik_sim/*.py` 全部使用裸 `print`，没有跨模块共享的 logger 实例；这与硬件侧的 `RunLogger` 形成对比。
- **FlashSAC 算法层不产日志**：`hardware/flash_rl/` 下零处出现 `import logging`，算法通过 `hooks.last_info` 向训练循环暴露标量指标，由 `train.py` 决定如何记录。

## 5. 总结

PolicyIK 的日志系统是“轻量级自实现”风格：硬件训练管线通过 `RunLogger` 把结构化指标（JSONL）与控制台输出（`console.log`）解耦，仿真管线则完全依赖 `print` 与外部 TensorBoard。仓库没有统一的日志框架、没有日志级别、也没有跨模块的 logger 单例——这是一个面向实验脚本的实用型设计。