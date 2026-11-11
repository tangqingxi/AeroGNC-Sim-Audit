# 四种 Reference 的统一推力链

DEV `3acba790adcaad8215eccb69f495b5f89d8a435b`。

| Reference | T 来源 | 上游 Topic | 下游路径 |
|---|---|---|---|
| Position | Velocity PID | control/attitude_setpoint | Attitude → rate_setpoint → Rate → thrust_setpoint → Allocation |
| Velocity | Velocity PID | control/attitude_setpoint | Attitude → rate_setpoint → Rate → thrust_setpoint → Allocation |
| Attitude | Guidance 固定参考 | guidance/attitude_reference | Attitude → rate_setpoint → Rate → thrust_setpoint → Allocation |
| BodyRate | Guidance 固定参考 | guidance/rate_reference | Rate → thrust_setpoint → Allocation |

rate_setpoint 为 omega(rad/s)、alpha(rad/s²)、thrust_n(N)；Rate 独立发布 torque(N·m) 与 thrust_n(N)，同一次更新、同 sample_time。
推力传递不是推力控制；Attitude/Rate 原样转发，不重算、不限幅、不归一化。
Allocation 只接收两个控制输出，缺失或过期不产生新分配，既有命令保持；物理限幅在 Allocation 内保留。
BodyRate 固定推力默认 trim.thrust_n，装配范围来自实际电机 k_T0·omega_range²。

WTR 保持 windtrace.flight.v1 / windtrace.parameters.v1；vehicle_rates_setpoint 仍是角速度。
vehicle_thrust_setpoint.thrust_n 使用实际 Rate 日志采样时间，16-topic WTR 由现有 WindTrace 通用字段列表自动读取和绘图。
四链逐级数值与因果验证见 test_thrust_setpoint_chain；历史逐点零差异见 evidence/closed-loop-comparison.json。
Physical Validation BLOCKED。
