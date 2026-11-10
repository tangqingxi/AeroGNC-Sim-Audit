# Experiment Definition

## 1. Schema

`cfg.experiment`：

```text
cfg.experiment
├── schema = 'aero-experiment-definition/1'
├── definition
│   ├── scenario
│   ├── method
│   ├── active_modules
│   ├── guidance
│   ├── navigation
│   ├── controllers
│   ├── allocation
│   ├── rates
│   └── components
├── definition_sha256
└── identity_sha256
```

`definition` 只包含纯值：数值、字符、逻辑、struct、cell；不含函数句柄、对象、
绝对路径、输出目录或 wall_time。

## 2. Scenario / Guidance

HoverHold：

```text
scenario = hover_hold
method   = P_PID_P_PID_PINV
active_modules = guidance, position, velocity, attitude, rate, allocation
guidance.position_m, velocity_m_s, acceleration_m_s2, yaw_rad, yaw_rate_rad_s
```

BodyRate Step：

```text
scenario = rate_step
method   = RATE_PID_PINV
active_modules = guidance, rate, allocation
guidance.axis, amplitude_rad_s, start_s, end_s, alpha_ff_rad_s2
```

全部 SI。

## 3. Navigation

```text
estimator = IdealState
ideal = true
measurement_source = sensor/state
acceleration_method = finite_difference
acceleration_limit_m_s2 = 50
uses_truth_derived_state = true
```

明确是测试适配器，不是 EKF；不含噪声/偏置模型。

## 4. Controllers

| 级 | 记录字段 |
|---|---|
| Position | Kp、velocity_limit_xy/up/down |
| Velocity | Kp/Ki/Kd、integral_limit、max/min/hover thrust、horizontal margin、max_tilt |
| Attitude | Kp、rate_limit、yaw_weight、ref_model 的 enabled/omega_n/ff_gain/ff_limit |
| Rate | Kp/Ki/Kd/Kff、integral_limit、saturation_tolerance |

只有 active_modules 中的控制器出现在 `definition.controllers`。运行时回调从
同一字段构造，不另造 gains。

## 5. Allocation

```text
algorithm = PINV
B_M、B_tau（trim 处名义 N·m/rad）
l1_m、l2_m、servo_torque_coefficient
trim_delta_rad、delta_limit_rad
ve_min_m_s、hover/max/min thrust
k_T0、motor_min/max_rad_s、motor_model='Omega=sqrt(T/k_T0)'
saturation_tolerance
```

电机与六舵面继续分离，无七维统一求解器。

## 6. Components

`definition.components` 是 `aero_validate_config` 之后的实际调度表：

```text
name, kind, period, priority, offset, delay, latches, topic
```

明确不包含 `step`/`sample` 函数句柄。Sensor/Navigation/Guidance/Position/
Velocity/Attitude/Rate/Allocation/Servo Latch/Motor Latch 的实际字段由这里恢复。

## 7. Hash

```text
definition_sha256 = SHA256(jsonencode({
  schema: 'aero-experiment-definition/1',
  definition: definition
}))
```

`definition_sha256` 不包含自身。`identity_sha256` 计算：

```text
identity = {
  schema: 'aero-experiment-identity/1',
  experiment: {schema, definition, definition_sha256},
  runtime: {tspan, x0, initial_command, seed, solver, environment,
            time_tolerance, components, communication},
  model: result.model_metadata
}
identity_sha256 = SHA256(jsonencode(identity))
```

`identity_sha256` 不包含 run_id/output_root/wall_time，因此同一实验输入得到同一
签名；改变 seed/tspan/x0/solver/environment/组件/模型/控制参数都会改变签名。

## 8. 验证

`evidence/experiment-definition-validation.txt`：

- schema/scenario/method/modules 正确；
- 相同配置重复构造 → 定义/身份 SHA 相同；
- 修改 yaw → 定义 SHA 改变；
- 无函数/对象句柄；
- 结果保存的 experiment 与构造时相同。
