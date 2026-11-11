# Velocity-Reference-Scenario — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `Velocity-Reference-Scenario` |
| 状态 | `in-progress (ready-for-review)`（未合并 main，未创建 Tag） |
| DEV 分支 | `feat/velocity-reference-scenario` @ `f5fab46079a197432408d653d3c32093dc3b6f61` |
| DEV 起点 | `main` @ `31159589f743b205f68b5777a70a24377dc57723` |
| AUDIT 分支 | `study/velocity-reference-scenario`（自 `9e3f220dd3f90402a120a772fa2f2bcaa56aed81`） |
| 测试 | 48/48 通过（全新 clone，89.83 s） |
| 物理验证 | **BLOCKED**（合成气动 fixture） |

## 1. 完成内容

新增第一个独立速度参考闭环场景：

```text
Guidance velocity_step
  → guidance/velocity_reference
  → Velocity PID
  → Attitude P
  → Rate PID
  → PINV Allocation
  → SHC09
```

关键实现：

1. `src/GNC/Guidance/velocity_step.m`：分段常值速度参考，输出
   `guidance/velocity_reference`，载荷 `velocity`、`acceleration_ff`、`yaw`、
   `yaw_rate`；不做速度跟踪、PID 或姿态计算。
2. `src/Scenarios/velocity_step_scenario.m`：默认速度 `[0.8;0;0] m/s`、
   `0.5–2.5 s`、`tspan [0,5]`；从悬停配平初态开始。
3. `build_config` 增加 `velocity` 模式：组件为 sensor/navigation/guidance_velocity/
   velocity/attitude/rate/allocation/latches，**不存在 position**。
4. Velocity PID 直接订阅 `guidance/velocity_reference`；HoverHold 仍使用
   `control/velocity_setpoint`。
5. Experiment Definition：`scenario=velocity_step`、
   `method=VEL_PID_P_PID_PINV`、active_modules=guidance/velocity/attitude/rate/
   allocation。
6. `topic_specs` 注册速度参考 Topic（3×1 m/s、3×1 m/s²、1×1 rad、1×1 rad/s），
   并加入实际发布日志。
7. WTR `trajectory_setpoint` 的 velocity/acceleration 来源改为
   `guidance/velocity_reference`；position 明确为
   `not_applicable (velocity-only scenario; no position reference)` 的零占位。
8. parameters.json 增加速度参考六项参数。
9. README/DESIGN 更新 Velocity Reference 状态、控制链和订阅关系。

## 2. 实际运行指标

默认 fixture、默认增益、`tspan [0,5]`：

| 指标 | 结果 |
|---|---:|
| 最大速度 | 0.722858 m/s |
| 阶跃期间北向速度 RMSE | 0.443423 m/s |
| 结束参考后残余速度 | 0.016233 m/s |
| 最大东向速度 | 0.016718 m/s |
| 最大下向速度 | 0.044851 m/s |
| 位置累计位移 | 1.472493 m |
| 最大倾角 | 5.394005° |
| 最大总推力 | 20.837082 N |
| 最大电机转速 | 848.182251 rad/s |
| 最大舵面偏角 | 0.184307 rad |
| 分配饱和计数 | 0 |
| 日志丢弃 | 0 |
| 状态有限 | 是 |

本场景没有 Position 控制器，速度回到零不代表位置回到原点；位置累计位移是
预期结果，不使用悬停位置保持指标评价。

## 3. 绕过 Position 的证据

- 组件表不含 `position`；
- `active_modules` 不含 position；
- `definition.controllers` 不含 position；
- 发布日志中 `position` 记录数 0；
- 发布日志中 `control/velocity_setpoint` 记录数 0；
- 速度参考记录 250 条；
- `method=VEL_PID_P_PID_PINV`；
- WTR `trajectory_setpoint.position` 来源明确为 not applicable。

证据：[control-chain.txt](evidence/control-chain.txt)。

## 4. Guidance 验证

`guidance-reference.txt` 记录：

- `t=0.49`：零速度；
- `t=0.50`：`[0.8;0;0]`、前馈 `[0.1;-0.2;0.05]`、yaw `0.3`、yaw_rate `0.1`；
- `t=2.40`：仍为目标速度；
- `t=2.50`：恢复零速度；
- Topic 字段、尺寸、单位符合契约。

## 5. 测试

```text
全新 clone feat/velocity-reference-scenario @ f5fab46
matlab -batch "run('tests/run_tests.m')"
→ 48/48 通过，89.83 s，退出码 0
```

原有 45 项测试全部保留（逐项比对 removed=空）；新增：

- `test_velocity_step_guidance`；
- `test_velocity_step_chain`；
- `test_velocity_step_closed_loop`。

## 6. WTR 与 WindTrace

- WTR v1 schema 未改；
- `wtr_validate` 通过；
- `trajectory_setpoint.velocity` 来自 `guidance/velocity_reference`；
- 15 topics，106 parameters；
- WindTrace `read_flight` 与 `--smoke` 成功；
- 离屏截图成功；
- 参数包含六项速度参考追溯字段。

证据：[wtr-validation.json](evidence/wtr-validation.json)、
[windtrace-smoke.txt](evidence/windtrace-smoke.txt)。

## 7. 未解决/限制

1. 物理验证 **BLOCKED**：fixture 不是真实气动/实机数据。
2. 未做 PX4 SITL 并排对照。
3. `IdealState` 没有传感器噪声/偏置，不是 EKF。
4. 最大速度约 0.723 m/s（参考 0.8），受现有增益/气动/限幅影响；本轮未调参。
5. 未实现 Attitude Reference；未修改 Velocity PID、Position/Attitude/Rate 算法、
   PINV、Aircraft、执行器、Simulation、WTR 或 WindTrace。
6. 速度场景的位置会持续漂移；这是无位置环的预期行为。

## 8. 提交

| 提交 | 内容 |
|---|---|
| `f5fab46` | 独立速度参考场景、Guidance、Topic、Experiment Definition、WTR 与测试 |

完整分支审查应覆盖 `main...feat/velocity-reference-scenario`；本阶段只描述相对
`3115958` 的增量。

## 9. 结论

Velocity PID 可独立接收 Guidance 速度参考；Position P 真正不参与；下游
Attitude/Rate/PINV 完整复用；MAT/WTR/WindTrace 可记录并回放。可以提交
NEXUS 复核，物理验证仍为 BLOCKED。
