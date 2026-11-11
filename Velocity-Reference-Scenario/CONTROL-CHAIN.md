# Control Chain — Velocity Reference

## 1. 数据流

```text
Guidance velocity_step
  publish guidance/velocity_reference
    velocity (NED, m/s)
    acceleration_ff (NED, m/s²)
    yaw (rad)
    yaw_rate (rad/s)
        │
        ▼
Velocity PID
  setpoint_topic = guidance/velocity_reference
  nav_topic      = navigation/state
        │
        ▼
control/attitude_setpoint（q, thrust_n, yaw, yaw_rate）
        │
        ▼
Attitude P
  output control/rate_setpoint（omega, alpha）
        │
        ▼
Rate PID
  output control/torque_setpoint
        │
        ▼
PINV Allocation
  output control/allocation_status 与 command/computed
        │
        ▼
Servo/Motor latch → SHC09
```

## 2. 组件

```text
sensor_ideal
navigation
guidance_velocity (50 Hz)
velocity (100 Hz)
attitude (200 Hz)
rate (400 Hz)
allocation (400 Hz)
latch_servos (200 Hz)
latch_motor (50 Hz)
```

**不存在** `position` 组件。没有用空运行模块或零误差 Position 来伪造独立性。

## 3. Topic 契约

`guidance/velocity_reference` 在
`src/GNC/Control/Internal/topic_specs.m` 唯一定义：

| 字段 | 尺寸 | 单位 |
|---|---|---|
| `velocity` | 3×1 | m/s |
| `acceleration_ff` | 3×1 | m/s² |
| `yaw` | 1×1 | rad |
| `yaw_rate` | 1×1 | rad/s |

该 Topic 加入实际发布日志列表。

HoverHold 仍使用：

```text
guidance/reference → Position → control/velocity_setpoint → Velocity
```

Velocity Step 使用：

```text
guidance/velocity_reference → Velocity
```

两条路径在 Attitude P 汇合。

## 4. 时序与因果

- Guidance 50 Hz（period 0.020 s）；
- Velocity 100 Hz（0.010 s）；
- Attitude 200 Hz（0.005 s）；
- Rate/Allocation 400 Hz（0.0025 s）；
- 下游只读取已发布上游输出；
- 测试检查所有日志 `available_time_s >= sample_time_s`，不存在未来数据泄漏。

## 5. 实际绕过的证据

`result.config.components`：

```text
sensor_ideal, navigation, guidance_velocity, velocity, attitude, rate,
allocation, latch_servos, latch_motor
```

`result.config.experiment.definition.active_modules`：

```text
guidance, velocity, attitude, rate, allocation
```

发布日志：

```text
guidance/velocity_reference records = 250
position records = 0
control/velocity_setpoint records = 0
```

`method = VEL_PID_P_PID_PINV`。

证据：[`evidence/control-chain.txt`](evidence/control-chain.txt)。
