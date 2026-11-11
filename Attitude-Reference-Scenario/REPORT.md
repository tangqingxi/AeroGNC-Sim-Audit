# Attitude-Reference-Scenario — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `Attitude-Reference-Scenario` |
| 状态 | `in-progress (ready-for-review)`（未合并 main，未创建 Tag） |
| DEV 分支 | `feat/attitude-reference-scenario` @ `abcc57b20eac3194fea520e6db5b1199ee0cd183` |
| DEV 起点 | `main` @ `f0890c2434d9734693a08b488998481e26554486` |
| AUDIT 分支 | `study/attitude-reference-scenario`（自 `4fbce98b372e2616b1b591408581058b481e514e`） |
| 测试 | 51/51 通过（全新 clone，255.06 s） |
| 物理验证 | **BLOCKED**（合成气动 fixture） |

## 1. 完成内容

新增独立姿态参考闭环：

```text
Guidance attitude_step
  → guidance/attitude_reference
  → Attitude P
  → Rate PID
  → PINV Allocation
  → SHC09
```

实现：

1. `src/GNC/Guidance/attitude_step.m`：分段常值姿态参考，不做姿态跟踪、
   角速度计算或执行器控制。
2. `src/GNC/Guidance/euler_to_quat_bn.m`：3-2-1 `R_bn=Rz(yaw)Ry(pitch)Rx(roll)`，
   通过项目现有 `rotation_to_quat` 生成标量在前 Hamilton 四元数；不另建坐标约定。
3. `src/Scenarios/attitude_step_scenario.m`：默认 +5° 滚转、0.5–1.5 s、
   `[0,3]` s、固定 `trim.thrust_n`。
4. `build_config` 增加 `attitude` 模式：sensor/navigation/guidance_attitude/
   attitude/rate/allocation/latches；**无 position、无 velocity**。
5. Attitude P 直接订阅 `guidance/attitude_reference`；Allocation 从同一 Topic
   读取固定 `thrust_n`；Rate/PINV/执行器复用。
6. Experiment Definition：`scenario=attitude_step`、
   `method=ATT_P_RATE_PID_PINV`、active_modules=guidance/attitude/rate/allocation。
7. Topic `guidance/attitude_reference`：`quaternion` 4×1、`thrust_n` N、
   `yaw` rad、`yaw_rate` rad/s。
8. WTR `vehicle_attitude_reference`/`vehicle_attitude_setpoint` 使用 Guidance
   姿态参考；`trajectory_setpoint` 明确 not_applicable；parameters.json 增加
   八项姿态参考追溯。
9. README/DESIGN 更新 Attitude Reference 状态与固定推力限制。

## 2. 姿态跟踪结果

默认 5° 滚转阶跃：

| 指标 | 结果 |
|---|---:|
| 90% 上升时间 | 0.470 s |
| 最大滚转 | 5.357698° |
| 超调 | 0.357698° |
| 阶跃期间最大姿态误差 | 5.000000°（阶跃瞬间） |
| 稳态滚转误差（1.0–1.5 s） | 0.244797° |
| 恢复滚转 <0.5° 用时 | 0.466 s |
| 最终滚转 | 0.017981° |
| 最终俯仰 | 0.020090° |
| 最终整体姿态误差 | 2.021983° |
| 最终 yaw | -2.021800° |
| 最大角速度模长 | 0.259079 rad/s |

解释：

- 滚转跟踪与恢复良好，稳态滚转误差约 0.245°；
- 阶跃瞬间“最大姿态误差 5°”是参考跳变时的瞬时误差，不是稳态误差；
- 结束时整体四元数误差约 2.02°，主要来自 yaw 约 -2.02° 漂移；该场景
  没有位置/速度/yaw 积分外环，固定配平与气动会产生偏航漂移。未调参掩盖。
- 无 Position/Velocity，姿态倾斜产生平动：
  - 位置累计位移 1.245165 m；
  - 最大速度 0.764767 m/s；
  这是预期物理行为，不作为位置保持验收指标。

## 3. 固定推力与执行器

| 指标 | 结果 |
|---|---:|
| 固定推力参考 | 20.554800 N（trim） |
| WTR 记录的 thrust 最小/最大 | 20.554800 / 20.554800 N |
| 最大电机转速 | 842.417458 rad/s |
| 最大舵面偏角 | 0.182897 rad |
| 分配饱和计数 | 0 |
| 状态有限 | 是 |
| 日志丢弃 | 0 |

电机转速由 `Ω=sqrt(T_ref/k_T0)` 独立求解，六舵面仍由 PINV 分配。

## 4. 绕过 Position/Velocity 的证据

- 组件不含 `position`、`velocity`；
- `active_modules` 不含 position/velocity；
- `definition.controllers` 不含 position/velocity；
- 日志 `position`、`velocity` 记录均为 0；
- `control/velocity_setpoint` 记录 0；
- `control/attitude_setpoint` 记录 0；
- Attitude P 直接读 `guidance/attitude_reference`；
- `control/rate_setpoint` 599 条；
- `guidance/attitude_reference` 150 条。

证据：[control-chain.txt](evidence/control-chain.txt)。

## 5. 测试

```text
全新 clone feat/attitude-reference-scenario @ abcc57b
matlab -batch "run('tests/run_tests.m')"
→ 51/51 通过，255.06 s，退出码 0
```

原有 48 项全部保留（逐项比对 removed=空）；新增：

- `test_attitude_step_guidance`；
- `test_attitude_step_chain`；
- `test_attitude_step_closed_loop`。

测试覆盖四元数/角度阶跃、Topic 契约、无 Position/Velocity、固定推力、
多速率/因果、WTR 参数、MAT roundtrip、姿态响应/恢复/执行器边界。

## 6. WTR 与 WindTrace

- WTR v1 schema 未改；
- 15 topics、99 parameters；
- `vehicle_attitude_reference.q` 与 `vehicle_attitude_setpoint.q` 来源为
  `guidance/attitude_reference (attitude-only scenario)`；
- `vehicle_rates_setpoint.xyz` 来源为 `control/rate_setpoint`；
- `trajectory_setpoint.velocity/position` 明确 not_applicable；
- `wtr_validate` 通过；
- WindTrace `read_flight`、`--smoke` 为 completed；
- 离屏截图成功。

证据：[wtr-validation.json](evidence/wtr-validation.json)、
[windtrace-smoke.txt](evidence/windtrace-smoke.txt)。

## 7. 未解决/限制

1. 物理验证 **BLOCKED**：fixture 不是真实气动/实机数据。
2. 未做 PX4 SITL 并排对照。
3. IdealState 无传感器噪声/偏置，不是 EKF。
4. 固定推力，不是推力控制算法；不能描述为完整悬停控制。
5. 无位置/速度环，位置持续漂移是预期结果。
6. 结束时 yaw 约 -2.02° 漂移；未被调参掩盖，作为本场景限制记录。
7. 未修改 Attitude P/Rate PID/PINV/Aircraft/执行器/Simulation/WTR/WindTrace。

## 8. 提交

| 提交 | 内容 |
|---|---|
| `abcc57b` | 独立姿态参考场景、Guidance、Topic、Experiment Definition、WTR 与测试 |

完整分支审查应覆盖 `main...feat/attitude-reference-scenario`；本阶段只描述相对
`f0890c2` 的增量。

## 9. 结论

Attitude P 可在无 Position/Velocity 的情况下独立接收 Guidance 姿态参考；
Rate PID、PINV 和执行器完整复用；MAT/WTR/WindTrace 可记录并回放。滚转跟踪
与恢复通过，yaw 漂移与物理限制已如实记录。可以提交 NEXUS 复核。
