# Scenario Design — Velocity Step

## 1. 场景接口

```matlab
source = struct('type', 'fixture', 'tables', shc09_aero_fixture());
scenario = velocity_step_scenario(struct('aero_source', source));
result = aero_sim(scenario.model, scenario.cfg);
```

输出 `s`：`name`、`mode`、`model`、`cfg`；唯一实验定义在
`s.cfg.experiment`，不返回独立 gains/signature/reference 副本。

## 2. 默认参数

| 参数 | 默认值 | 单位 |
|---|---:|---|
| `velocity_reference` | `[0.8; 0; 0]` | m/s，NED |
| `start_s` | 0.5 | s |
| `end_s` | 2.5 | s |
| `tspan` | `[0, 5]` | s |
| `yaw` | 0 | rad |
| `yaw_rate` | 0 | rad/s |
| `acceleration_ff` | `[0; 0; 0]` | m/s²，NED |
| `max_records` | 300000 | 条 |

速度方向是 NED 北向，不是机体 FRD 前向。

## 3. 合法性检查

- `velocity_reference`/`acceleration_ff` 必须是有限 3×1；
- `tspan` 必须有限、递增；
- `tspan(1) <= start_s < end_s <= tspan(2)`；
- `yaw`/`yaw_rate` 必须是有限标量；
- 速度参考在 `t >= end_s` 后恢复零。

## 4. Experiment Definition

```text
scenario = velocity_step
method   = VEL_PID_P_PID_PINV
active_modules = guidance, velocity, attitude, rate, allocation

guidance:
  type = VelocityStep
  output_topic = guidance/velocity_reference
  velocity_reference_ned_m_s
  acceleration_ff_ned_m_s2
  yaw_rad
  yaw_rate_rad_s
  start_s / end_s

controllers:
  velocity (Kp/Ki/Kd/integral_limit/max-min-hover thrust/margin/tilt)
  attitude (Kp/rate_limit/yaw_weight/ref_model)
  rate (Kp/Ki/Kd/Kff/integral_limit/saturation_tolerance)

allocation:
  PINV、B_M/B_tau、l1/l2、trim、delta limits、ve_min、
  hover/max/min thrust、motor limits、saturation tolerance

navigation:
  IdealState（非 EKF）
```

没有 `controllers.position`，没有 position component。

## 5. 初值与物理边界

- 从 `shc09_hover_equilibrium` 的悬停配平初态开始；
- 使用 SHC09 非线性六自由度与现有执行器；
- `fixture` 只用于软件通路和数值行为，不是物理验证。

## 6. 场景输出评价

这是速度参考场景，不评价位置回原点。评价重点是：

- 北向速度响应与 RMSE；
- 撤销参考后的残余速度；
- 倾角、推力、舵面、电机与饱和；
- 日志完整性、状态有限性；
- 无 Position 组件和 setpoint 记录。
