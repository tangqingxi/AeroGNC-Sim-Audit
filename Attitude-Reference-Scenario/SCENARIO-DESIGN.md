# Scenario Design — Attitude Step

## 1. 场景接口

```matlab
source = struct('type', 'fixture', 'tables', shc09_aero_fixture());
s = attitude_step_scenario(struct('aero_source', source));
result = aero_sim(s.model, s.cfg);
```

输出 `s`：`name`、`mode`、`model`、`cfg`；唯一实验定义在 `s.cfg.experiment`。

## 2. 默认参数

| 参数 | 默认值 | 单位 |
|---|---:|---|
| `roll_rad` | +5°（`deg2rad(5)`） | rad |
| `pitch_rad` | 0 | rad |
| `yaw_rad` | 0 | rad |
| `yaw_rate` | 0 | rad/s |
| `thrust_reference_n` | `trim.thrust_n` | N |
| `start_s` | 0.5 | s |
| `end_s` | 1.5 | s |
| `tspan` | `[0, 3]` | s |
| `max_records` | 200000 | 条 |

参考行为：

```text
t < 0.5 s:          level
0.5 <= t < 1.5 s:   roll = +5°
t >= 1.5 s:         level
```

## 3. 合法性检查

- roll/pitch/yaw/yaw_rate 必须是有限标量；
- `tspan` 有限递增；
- `tspan(1) <= start_s < end_s <= tspan(2)`；
- `thrust_reference_n` 必须在电机实际推力范围
  `[k_T0·Ω_min², k_T0·Ω_max²]`；
- 默认推力为 `model.trim.thrust_n`。

## 4. 四元数约定

使用项目现有：

- `rotation_to_quat.m`
- `quat_to_rotation.m`

3-2-1 Euler：

```text
R_bn = Rz(yaw) * Ry(pitch) * Rx(roll)
```

`euler_to_quat_bn.m` 只构造 `R_bn` 并调用 `rotation_to_quat`，输出标量在前
Hamilton 四元数，表示 FRD→NED。

## 5. Experiment Definition

```text
scenario = attitude_step
method   = ATT_P_RATE_PID_PINV
active_modules = guidance, attitude, rate, allocation

guidance:
  type = AttitudeStep
  output_topic = guidance/attitude_reference
  roll_rad / pitch_rad / yaw_rad
  quaternion_bn / quaternion_level_bn
  thrust_reference_n
  yaw_rate_rad_s
  start_s / end_s

controllers:
  attitude (Kp/rate_limit/yaw_weight/ref_model)
  rate (Kp/Ki/Kd/Kff/integral_limit/saturation_tolerance)

allocation:
  PINV、B_M/B_tau、l1/l2、trim、delta limits、ve_min、
  hover/max/min thrust、motor limits、saturation tolerance
```

没有 position/velocity controllers。

## 6. 固定推力与平动

- `thrust_reference_n` 是固定总推力参考，不是推力控制器；
- 姿态倾斜产生 NED 平动加速度和位置漂移；
- 场景没有 Position/Velocity 环，不能用位置保持指标评价；
- 电机推力仍由 `Ω=sqrt(T_ref/k_T0)` 独立求解，六舵面由 PINV 分配力矩。

## 7. 验收重点

- 姿态参考前/中/后与四元数单位化；
- Position/Velocity 组件与发布记录均不存在；
- Attitude P 直接订阅 Guidance；
- Rate/PINV 正常；
- 固定推力对应电机转速；
- 多速率与因果；
- WTR 姿态参考来源和参数；
- 状态/执行器边界、日志完整性。
