# Control Chain — Attitude Reference

## 1. 数据流

```text
Guidance attitude_step
  publish guidance/attitude_reference
    quaternion (q_bn, scalar-first Hamilton)
    thrust_n   (N, fixed reference)
    yaw        (rad)
    yaw_rate   (rad/s)
        │
        ├──────────────► Attitude P
        │                  setpoint: guidance/attitude_reference
        │                  output:   control/rate_setpoint
        │
        └──────────────► PINV Allocation
                           thrust_n from guidance/attitude_reference
                           torque from control/torque_setpoint
```

## 2. 组件

```text
sensor_ideal
navigation
guidance_attitude (50 Hz)
attitude (200 Hz)
rate (400 Hz)
allocation (400 Hz)
latch_servos (200 Hz)
latch_motor (50 Hz)
```

**不存在** `position` 或 `velocity` 组件。没有空运行模块。

## 3. Topic 契约

`guidance/attitude_reference` 在唯一的
`src/GNC/Control/Internal/topic_specs.m` 注册：

| 字段 | 尺寸 | 单位 |
|---|---|---|
| `quaternion` | 4×1 | 1 |
| `thrust_n` | 1×1 | N |
| `yaw` | 1×1 | rad |
| `yaw_rate` | 1×1 | rad/s |

## 4. 复用关系

| 模块 | 输入 | 输出 | 修改 |
|---|---|---|---|
| Attitude P | nav + guidance attitude | rate setpoint | 仅配置 topic；控制律未改 |
| Rate PID | rate setpoint + allocation status | torque setpoint | 未改 |
| PINV | torque + guidance thrust | servo/motor | 未改；thrust 来自 guidance |
| Servo/Motor latch | command | held command | 未改 |
| SHC09 | command | state | 未改 |

## 5. 绕过 Position/Velocity 的证据

组件表不含 position/velocity；Experiment Definition 的 active_modules 只有
guidance/attitude/rate/allocation；发布日志：

```text
position records = 0
velocity records = 0
control/velocity_setpoint records = 0
control/attitude_setpoint records = 0
guidance/attitude_reference records = 150
control/rate_setpoint records = 599
```

`method = ATT_P_RATE_PID_PINV`。

## 6. 固定推力

```text
thrust_reference_n = 20.554800 N (trim)
allocation motor  = sqrt(T_ref / k_T0)
```

WTR guidance 记录中 thrust 最小/最大均为 20.554800 N；没有推力闭环。

## 7. 时序与因果

- Guidance 50 Hz（0.020 s）；
- Attitude 200 Hz（0.005 s）；
- Rate/Allocation 400 Hz（0.0025 s）；
- 下游只读取已发布上游输出；
- 测试检查所有日志 `available_time_s >= sample_time_s`。

证据：[`evidence/control-chain.txt`](evidence/control-chain.txt)。
