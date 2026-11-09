# GNC-Baseline — 源码研究与复用分类

## 参考版本（只读，未复制第三方代码）

| 仓库 | 版本 | 用途 |
|---|---|---|
| DuctedFanUAV-Autopilot | `df-main` @ `2eb33667575619b02a2d9b52032df2fac7fb5291` | P–PID–P–PID + PINV 主参考 |
| Ducted-Fan-SHC09-matlab | `43b6c7aef1912f8118e54d3decac1ac55465d5b1` | SHC09 控制/分配/场景历史实现 |
| WindTrace | `bcdb6a8eca9334b395759c604d5c3978feb1bd8f` | `windtrace.flight.v1` 读取器与 Viewer |

阅读的 PX4 文件：`PositionControl.cpp`、`AttitudeControl.cpp`、`rate_control.cpp`、
`ControlAllocationPseudoInverse.cpp`、`MulticopterRateControl.cpp`、`ControlAllocator.cpp`
及其头文件/参数定义。阅读的旧 MATLAB 文件：`ca_position_control.m`、`ca_attitude_control.m`、
`example_pinv_allocate.m`、`export_windtrace_flight.m`、平台模块参数入口。

## 复用分类

| 功能 | 分类 | 说明 |
|---|---|---|
| 位置 P → 速度指令 | ADAPT | 采用 PX4/旧项目的误差乘增益语义；单位为 SI |
| 速度 PID + 前馈 + 限幅/抗饱和 | ADAPT | 保留 P/I/前馈、积分限幅与饱和冻结；未照搬 NaN 缺省语义 |
| 推力矢量 → 姿态（thrustToAttitude） | ADAPT | NED/FRD 下 `T=\|m(a−g)\|`、`z_b=−F/\|F\|`，增加倾角限制 |
| 姿态四元数误差 + P | ADAPT | 采用 `2·vec(q⁻¹⊗q_d)`（quaternion-imag 语义），不含参考模型 |
| 角速度 PID（P/I/D+FF、积分限幅、饱和门控） | ADAPT | 使用 SI 力矩；D 用导航角加速度，未使用 PX4 的 i_factor |
| 伪逆分配（trim + pinv + 限幅） | ADAPT | 使用物理 `B_tau`；不做 PX4 归一化矩阵缩放 |
| PX4 INDI / CA16 / 参考模型 / 前馈调度 | DEFER | 纯 PID 基线明确不启用 |
| PX4 归一化力矩/推力链路 | REIMPLEMENT | 平台无归一化执行器空间，改为 N·m 与 N |
| 旧项目理想导航/传感器适配 | ADAPT | 改为 Topic `sensor/state → navigation/state`，标记 `ideal=1` |
| 旧项目 `example_pinv` | ADAPT | 保留“最小范数 + 限幅”思想，增加配平与残差反馈 |
| 旧项目 WindTrace 导出（Python 桥） | REIMPLEMENT | 用 MATLAB 直接写 `windtrace.flight.v1`，不依赖 Python 运行 |
| 旧项目 Scheduler/Bus/模块注册中心 | DEFER | 平台已有唯一 Topic Bus 与内核，不迁移旧框架 |

## 单位/坐标差异（已处理）

- PX4 角速率/力矩 setpoint 为归一化量；本平台 Rate PID 直接输出 N·m，分配器使用物理 `B_tau`。
- SHC09 已是 FRD/NED；导出层不再做旧项目的 SHC09→FRD 变换。
- 配平舵角非常数零：分配采用 `δ=δ_trim+pinv(B_tau)τ`，零力矩时保持配平，避免首拍打舵。
- 电机推力与舵面力矩分开：`Ω=sqrt(T/k_T0)`，不把七执行器混入同一 PINV。

## 未实现（明确）

PX4 的 INDI、CA16、姿态参考模型、速率前馈调度、执行器故障重构、控制分配归一化矩阵；
这些不属于本轮“纯 PID + PINV”基线。
