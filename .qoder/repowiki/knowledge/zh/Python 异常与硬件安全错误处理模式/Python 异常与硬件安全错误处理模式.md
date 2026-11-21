---
kind: error_handling
name: Python 异常与硬件安全错误处理模式
slug: error_handling
category: error_handling
scope:
    - '**'
---

## 总体方式

仓库是 Python + ROS 2 项目，没有自定义的 `errors/` 包或统一的错误类型体系。错误处理以 Python 内置异常（`ValueError`、`RuntimeError`、`FileNotFoundError`、`KeyError`、`TypeError`）为主，在硬件交互层辅以 `try/except KeyboardInterrupt` 和 `finally` 做资源清理；ROS/CAN 等外部子系统通过日志打印而非抛出异常来传递故障状态。

## 关键文件与位置

- `hardware/policyik_hw/robot/ros_io.py`：预检查 `check_single_control_stack()` 使用 `subprocess.check_output` 捕获 `Exception`，失败时打印警告并返回；成功但检测到双 controller_manager 或无 controller_manager 时抛出 `RuntimeError`。
- `hardware/policyik_hw/robot/motor_health.py`：CAN socket 读取循环用 `socket.timeout` 静默忽略超时；顶层 `KeyboardInterrupt` 后 `finally` 关闭 socket 并 shutdown rclpy。
- `hardware/policyik_hw/collect_demos.py`：采集主循环用 `try/except (KeyboardInterrupt, Exception)` 包裹，记录 `aborted` 原因，非 `RuntimeError` 时 `traceback.print_exc()`，`finally` 中 `env.close()` 并可能 `rclpy.shutdown()`。
- `hardware/policyik_hw/evaluate.py`：评估循环同样 `try/except KeyboardInterrupt`，中断时将当前 trial 加入 `skipped`，`finally` 保存结果并关闭环境。
- `hardware/policyik_hw/envs/reward.py`：参数校验集中抛 `ValueError`（如 `tau_p, tau_r_deg, tau_u, kp_size must be > 0`、`dt must be > 0`、`pbrs_gamma != agent gamma`）。
- `hardware/policyik_hw/rl/demos.py`：演示数据加载抛 `FileNotFoundError`（找不到文件）、`ValueError`（列缺失、观测维度不匹配、episode 不完整、接口不一致）。
- `hardware/flash_rl/agents/utils/network.py`：动态调用网络方法时校验 method 类型，抛 `ValueError` / `AttributeError` / `TypeError`。
- `hardware/flash_rl/buffers/torch_buffer.py`：缓冲区写入路径抛 `ValueError`。

## 架构与约定

1. **参数校验 → 内置异常**：所有配置/输入校验直接 raise `ValueError`，由上层 CLI 暴露给用户。例如 reward 配置、demo 数据结构、FlashSAC 网络构造。
2. **外部系统容错 → 打印 + 降级**：ROS 节点列表查询失败时只 `print(f"[preflight] could not list ROS nodes ({e}); skipping ...")` 并继续；motor health JSON 解析失败时 `except Exception: pass`，仅丢弃单条消息。
3. **长运行脚本 → KeyboardInterrupt + finally**：`collect_demos.py`、`evaluate.py`、`motor_health.py` 均捕获 `KeyboardInterrupt`，并在 `finally` 中确保关闭 CAN socket、ROS node、rclpy 等资源。
4. **原子写入**：`collect_demos.py` 写 `.tmp.npz` 后用 `os.replace` 原子替换；`evaluate.py` 写 `.tmp` JSON 后 `os.replace`，保证中断不会留下半截文件。
5. **健康状态 → 返回值而非异常**：`ArmIO.health_check()` 返回 `(bool, str)` 元组，将电机故障、温度超限、CAN 失联等状态编码为字符串原因，而不是抛异常——因为这是预期可达的运行期状态，不是编程错误。
6. **Motor 错误码映射**：`motor_health.py` 定义 `ERR_NAMES` 字典把 CAN 帧中的 ERR 字节映射为人类可读名称（disabled/enabled/overvoltage/undervoltage/overcurrent/MOS overtemp/rotor overtemp/comms lost/overload），未知码回退为十六进制字符串。
7. **运行时断言式错误**：`demos.py` 在 demo 数据不符合约束时抛 `ValueError("the demonstrations were collected under a different interface: ...")`，明确区分“数据格式错误”和“文件不存在”。

## 观察到的约定与约束

- 所有可恢复的外部 I/O 错误（ROS 子进程、CAN socket timeout、JSON 解析）采用 `try/except Exception` 捕获并转为日志/静默丢弃，不向上传播。
- 不可恢复的配置/契约违反（维度不匹配、缺少必需字段、负数参数）统一使用 `ValueError` / `FileNotFoundError` / `RuntimeError`。
- 长生命周期脚本（采集、评估、电机监控）一律用 `try/except KeyboardInterrupt` + `finally` 保证资源释放，未见到 `sys.exit` 或全局异常钩子。
- 没有自定义基类异常、错误码枚举或中间件式错误传播；错误边界集中在每个独立 CLI 脚本的 `main()` 顶部 try 块内。
- `ros_io.py` 的 `check_single_control_stack(raise_on_fail=True)` 是唯一一处按参数切换“抛异常 vs 打印警告”的模式，被调用方决定是否把预检失败视为致命错误。