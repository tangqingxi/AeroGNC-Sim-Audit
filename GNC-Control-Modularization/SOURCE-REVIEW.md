# GNC-Control-Modularization — 源码审查与文件映射

## 参考版本（只读）

| 仓库 | 版本 | 用途 |
|---|---|---|
| DuctedFanUAV-Autopilot | `df-main` @ `2eb33667575619b02a2d9b52032df2fac7fb5291` | 位置/速度/姿态/速率/PINV 语义 |
| Ducted-Fan-SHC09-matlab | `43b6c7aef1912f8118e54d3decac1ac55465d5b1` | SHC09 控制/分配/场景历史实现 |
| WindTrace | `bcdb6a8eca9334b395759c604d5c3978feb1bd8f` | `windtrace.flight.v1` 读取与渲染 |

本轮不新增第三方代码；只按 PX4 数据链语义重排自研控制链，全部控制量仍为 SI。

## 旧架构问题

1. `gnc_position_velocity_step.m` 同时实现 Position P 与 Velocity PID，二者共用
   一个组件、一个周期和一个函数作用域，无法独立替换/调参/测试。
2. 所有文件使用 `gnc_` 前缀，目录与文件名重复表达领域。
3. 姿态环把 `alpha_feedforward`（rad/s²）直接加到 `omega`（rad/s），量纲错误。
4. 速率环通过 `bus.gnc.allocation.saturated` 读取分配器内部状态，绕过 Topic 契约。
5. Velocity PID 的 `Kd` 参数未参与计算，也没有非零 Kd 测试。
6. WTR 的 PWM 时间轴使用 `result.time`，不是真实锁存事件时刻；参数与签名不完整；
   `.wtr` 写入未复用 Results 容量检查。

## 文件映射（Git 可追踪重命名）

| 旧文件/函数 | 新位置 |
|---|---|
| `Control/gnc_position_velocity_step.m` | `Control/Position/position_p.m` + `Control/Velocity/velocity_pid.m` |
| `Control/gnc_attitude_step.m` | `Control/Attitude/attitude_p.m` |
| `Control/gnc_rate_step.m` | `Control/Rate/rate_pid.m` |
| `Control/gnc_allocate_pinv_step.m` | `Control/Allocation/pinv_allocator.m` |
| `Guidance/gnc_hover_hold_step.m` | `Guidance/hover_hold.m` |
| `Guidance/gnc_body_rate_step.m` | `Guidance/body_rate_step.m` |
| `Navigation/gnc_ideal_state_step.m` | `Navigation/ideal_state.m` |
| `Control/gnc_latch_*_step.m` | `Scenarios/Internal/latch_*.m`（装配动作，不属于控制律） |
| `Control/Internal/gnc_*` | `Control/Internal/*`（无前缀） |
| `Scenarios/gnc_hover_scenario.m` | `Scenarios/hover_scenario.m` |
| `Scenarios/gnc_rate_step_scenario.m` | `Scenarios/rate_step_scenario.m` |
| `Scenarios/Internal/gnc_build_config.m` | `Scenarios/Internal/build_config.m` |

## 复用分类

| 功能 | 分类 | 说明 |
|---|---|---|
| PX4 位置 P / 速度 PID / 姿态误差 / 速率 PID | ADAPT | 数据链与限幅语义保留，增益改用 SI；D 使用测量加速度 |
| PX4 PINV（trim + pinv + 限幅 + 残差） | ADAPT | 物理 `B_tau`；明确残差/饱和定义 |
| PX4 INDI / CA16 / 参考模型 / 增益调度 | DEFER | 本轮仍为纯 PID 基线 |
| 旧 MATLAB 位置/姿态/PINV | ADAPT | 已在新平台重写，未迁移旧框架 |
| 旧 WindTrace Python 桥 | REIMPLEMENT | MATLAB 直写 v1 容器 |
| 旧 Scheduler/Bus/模块注册中心 | DEFER | 平台已有唯一 Topic Bus 与内核 |

## 结论

四级控制器的数学关系未变，变化在组织方式、状态归属、通信路径与 WTR 记录；
默认 Kd=0 时闭环结果与原基线逐位一致（见 `evidence/controller-comparison.txt`）。
